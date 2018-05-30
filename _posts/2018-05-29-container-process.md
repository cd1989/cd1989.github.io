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

容器内多进程指在一个容器中运行多个进程的情况（_一般并不推荐这种用法_）。虽然Docker鼓励“一个容器一个进程(one process per container)”的方式，然而，在有些情况下，我们不得不选择这种折衷的方案。比如一些传统的由多个进程紧密耦合的应用，就很难拆分到多个容器中。再如[docker-in-docker](https://github.com/jpetazzo/dind)的应用场景，我们需要再容器中启动docker deamon进程，同时运行自己的业务进程。

相较于单进程的情况，在容器中运行多进程将给开发者带来更多的挑战。这里我们将详细介绍一些常见问题的解决方法和注意事项。

## init进程

我们已经知道，Linux系统通过PID Namespace（空间）来隔离和管理进程。不同Namespace中的进程相互独立，可以有相同的进程号。进程以树的形式组织，其中进程号为1的进程（即init进程）作为根节点，有着特殊的地位。它是所有进程的父进程，能够“收养”子进程的“孤儿”进程并最终回收资源，结束进程。

内核还为init进程赋予了一项重要的特性——信号屏蔽。如果init进程中没有编写处理某个信号的代码逻辑，那么与init在同一个PID Namespace下的进程（即init的子进程）发送给它的该信号都会被屏蔽。这个功能的作用主要是防止init进程被误杀。

由于PID Namespace也是以树状结构组织，一个PID Namespace可以有子PID Namespace。对于父PID Namespace中进程发给init进程的信号，有两种不同的处理方式：

- 如果信号不是SIGKILL（强制销毁进程，当你执行`kill -9 PID`来销毁进程的时候发送的就是SIGKILL信号）或SIGSTOP（暂停进程），信号也会被忽略。
- 如果信号是SIGKILL或者SIGSTOP，子PID Namespace的init进程会强制执行该信号要求的动作（销毁或暂停），这样的主要目的是确保该进程能够被父PID Namespac销毁或暂停。

## 容器进程管理

在Docker中，进程的隔离和管理同样通过PID Namespace来实现。所有容器的进程都是Docker Deamon的子进程，不同容器的进程属于不同的PID Namespace。当容器启动的时候，容器会被分配一个PID Namespace，该Namespace是Docker Deamon所在Namespace的子节点。所以Docker Deamon能够对容器中的进程交互、监控和回收。

_每个容器都有自己的Namespace，那Init进程是哪个呢？_

容器的init进程就是容器的启动进程，也就是Entrypoint和CMD启动的进程。当init进程退出后，Docker会销毁对应的PID Namespace，并向容器内所有其它的子进程发送SIGKILL。

做为init进程，

