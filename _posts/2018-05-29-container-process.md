---
layout:     post
title:      "容器内多进程管理"
subtitle:   ""
date:       2018-05-29
author:     "Dylan Chen"
header-img: "img/post-bg-speed.jpg"
tags:
    - 进程管理
    - Graceful Shutdown
    - Container
    - Supervisor
---

容器内多进程指在一个容器中运行多个进程的情况（_一般并不推荐这种用法_）。虽然Docker鼓励“一个容器一个进程(one process per container)”的方式，然而，在有些情况下，我们不得不选择这种折衷的方案。比如一些传统的由多个进程紧密耦合的应用，就很难拆分到多个容器中。再如[docker-in-docker](https://github.com/jpetazzo/dind)的应用场景，我们需要在容器中启动docker deamon进程，同时运行自己的业务进程。

相较于单进程的情况，在容器中运行多进程将给开发者带来更多的挑战。这里我们将详细介绍一些常见问题的解决方法和注意事项。

## INIT进程

我们已经知道，Linux系统通过PID Namespace（空间）来隔离和管理进程。不同Namespace中的进程相互独立，可以有相同的进程号。进程采用树状结构管理，其中进程号为1的进程（即init进程）作为根节点，有着特殊的地位。它是所有进程的父进程，能够“收养”子进程的“孤儿”进程并最终回收资源，结束进程。

内核还为init进程赋予了一项重要的特性——信号屏蔽。如果init进程中没有编写处理某个信号的代码逻辑，那么与init在同一个PID Namespace下的进程（即init的子进程）发送给它的该信号都会被屏蔽。这个功能的作用主要是防止init进程被误杀。

PID Namespace也是以树状结构组织，对于父PID Namespace中进程发给init进程的信号，有两种不同的处理方式：

- 如果信号不是SIGKILL（强制销毁进程，例如`kill -9 PID`发送的就是SIGKILL信号）或SIGSTOP（暂停进程），信号也会被忽略。
- 如果信号是SIGKILL或者SIGSTOP，子PID Namespace的init进程会强制处理该信号，这样的主要目的是确保该进程能够被父PID Namespace销毁或暂停。

## 容器的INIT进程

Docker正是通过PID Namespace来隔离和管理进程的。不同容器拥有不同的PID Namespace，彼此独立，每个容器的启动进程就是对应的PID Namespace的init进程。

在Docker 1.11之前，每个容器的启动进程都是Docker Daemon的子进程，被Docker Daemon直接管理。在Docker 1.11之后，Docker Engine经过了重构，被拆分成了Dockerd，docker-containerd，docker-containerd-shim以及docker-runc四部分，每个容器的init进程被一个docker-containerd-shim进程管理。

```bash
$ docker run -d --name redis redis
f28fb057d5d57579f2e9a452625de96e64a893658dc9f268f4ef148207bb0659
$ docker exec redis ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
redis        1     0  0 10:36 ?        00:00:00 redis-server *:6379
root        18     0  0 10:41 ?        00:00:00 ps -ef
$ docker top redis
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
systemd+            9634                9617                0                   18:36               ?                   00:00:00            redis-server *:6379
```

从上面我们可以看到，在容器的PID Namespace中，启动的redis-server进程是init进程，而在宿主机上，该进程进程号为9634，父进程是9617，而这个9617进程就是创建该容器对应的docker-containerd-shim进程。

```
[root@c6v186 ~]# ps -ef | grep 9617
root      9617  2888  0 18:36 ?        00:00:00 docker-containerd-shim f28fb057d5d57579f2e9a452625de96e64a893658dc9f268f4ef148207bb0659 /var/run/docker/libcontainerd/f28fb057d5d57579f2e9a452625de96e64a893658dc9f268f4ef148207bb0659 docker-runc
systemd+  9634  9617  0 18:36 ?        00:00:04 redis-server *:6379
```

虽然我们知道容器的启动进程就是init进程，但是要确定到底那个是启动进程，我们还需要先弄清楚ENTRYPOINT和CMD的shell和exec两种模式。

#### SHELL vs EXEC 模式 

**exec模式**
在exec模式下，任务进程就是init进程。
```Dockerfile
FROM busybox
CMD [ "tail", "-f", "/dev/null" ]
```
```bash
$ docker build -t busybox-exec .
$ docker run -d busybox-exec
$ docker exec -it c166f14fd34b ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 tail -f /dev/null
    5 root      0:00 ps -ef
```

由于exec模式不会通过shell来执行命令，所以像`$PATH`这样的环境变量是无法获得的。

```Dockerfile
FROM busybox
CMD [ "echo", "$PATH" ]
```

```bash
$ docker run busybox-exec
$PATH
```

但可以通过一下方式来访问环境变量：

```Dockerfile
FROM busybox
CMD [ "/bin/sh", "-c", "echo $PATH" ]
```

**shell模式**
在shell模式下，Docker会以`/bin/sh -c "task command"`方式执行任务命令，init进程不是任务进程而是shell进程。
```
FROM busybox
CMD tail -f /dev/null
```

```bash
$ docker build -t busybox-shell .
$ docker run -d busybox-shell
$ docker exec -it 5104cec379fe ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 06:37 ?        00:00:00 /bin/sh -c tail -f /dev/null
root         6     1  0 06:37 ?        00:00:00 tail -f /dev/null
root         7     0  0 06:37 pts/0    00:00:00 ps -ef
```

在shell模式下，任务进程由shell进程启动，所以可以获取环境变量。

```Dockerfile
FROM busybox
CMD echo $PATH
```

```bash
$ docker run busybox-shell
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

## 容器内进程优雅退出

启动进程作为容器的init进程，它需要有基本的进程管理能力，正确处理某些系统信号，如SIGTERM，SIGINT，这样才能保证进程能够优雅地退出。在多进程的场景下，这点尤其重要。

例如，假设我们以exec模式启动以下的脚本作为容器的启动进程，

```bash
#!/bin/bash

/app &

wait $!
```

这里，为了演示方便，我们采用如下的脚本作为app，它能够捕捉SIGTERM，SIGINT信号，并且打印`Goodbye, App`消息。

```bash
#!/bin/bash

trap 'echo "Goodbye, App"; exit' TERM INT

sleep 500 &

wait $!
```

我们将得到两个进程：

- 运行脚本的shell进程
- app后台进程（其实该进程也包含一个子进程sleep 500）

其中shell进程是容器的init进程，由于它没有信号处理的逻辑，根据前面对init进程的描述，当我们企图停止这个容器（docker stop），shell进程将忽略收到的SIGTERM信号，这时作为子进程的app将无法收到该SIGTERM信号，因而无法执行必要的清理工作以优雅退出。当我们执行docker stop的时候，我们可以发现，容器最终是在等待10s后被Docker Daemon强制关闭，而不是优雅退出。

为了让进程app优雅退出，在上述例子中，最简答的办法就是使app成为init进程，这样它就能收到SIGTERM等信号。

```bash
#!/bin/bash

exec app
```

通过exec来执行app，将使得app进程替代当前的shell进程，所以最终只会存在一个app进程。这时，如果我们通过docker stop停止容器，将会看到如下输出并且容器立刻退出。

```
Goodbye, App
```

但是，如果容器内需要启动多个进程，上述方法将无能为力，这时我们需要通过`trap`来给启动脚本增加信号处理能力。

```bash
#!/bin/bash

/background-app &

exec /app
```

### 通过脚本管理多进程

_Dockerfile_
```Dockerfile
FROM debian:9.4

COPY ./graceful.sh /home
COPY ./app1.sh /home
COPY ./app2.sh /home

WORKDIR "/home"

CMD [ "./graceful.sh" ]
```

_graceful.sh_
```bash
#!/bin/sh

trap 'echo "Goodbye, Init"; kill -TERM $PID1 $PID2' TERM INT

/home/app1.sh &
PID1=$!

/home/app2.sh &
PID2=$!

wait $PID1
wait $PID2

EXIT_STATUS=$?
```

_app1.sh_
```bash
#!/bin/sh

trap 'echo "Goodbye, APP1"; kill $PID' INT TERM

echo "APP1 running"

sleep 500 &
PID=$!

wait $PID
```

_app2.sh_

```bash
#!/bin/sh

trap 'echo "Goodbye, APP2"; kill $PID' INT TERM

echo "APP2 running"

sleep 500 &
PID=$!

wait $PID
```

在graceful.sh中，我们通过trap捕捉到了SIGTERM，SIGINT信号，并且在收到信号后通过kill将SIGTERM信号传递给子进程app1，app2。

当运行上述容器时，

```bash
$ docker run graceful
APP1 running
APP2 running
```

而当我们通过docker stop停止容器时，

```bash
$ docker run graceful
APP1 running
APP2 running
Goodbye, Init
Goodbye, APP2
Goodbye, APP1
```

可以看到，app1，app2两个子进程均实现了优雅地退出。

在使用`trap`的时候，有一点需要尤其注意，

> When Bash receives a signal for which a trap has been set while waiting for a command to complete, the trap will not be executed until the command completes.

当Bash正在等待一个执行中的命令时，如果这时收到一个设置了trap的信号，在这个命令执行结束前trap不会被执行。这也是为什么之前的所有脚本中，进程都在后台运行（通过`&`），然后通过`wait`等待执行结束。

_**Refs:**_

- http://veithen.github.io/2014/11/16/sigterm-propagation.html

### 通过Supervisor管理多进程

Supervisor是一个方便的进程管理工具，通过它我们能够方便地在容器内启动多个进程，并且进程良好的管理。
