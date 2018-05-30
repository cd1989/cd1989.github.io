---
layout:     post
title:      "Docker架构演进之路"
subtitle:   ""
date:       2018-05-30
author:     "Dylan Chen"
header-img: "img/post-bg-speed.jpg"
tags:
    - Docker
    - 架构
    - Containerd
---

Docker主要由三部分组成：Docker Daemon，Docker Client以及Registry。Docker Daemon和Docker Client采用client-server的架构，如下图所示，Docker Daemon运行在宿主机上，Docker Client通过REST API与其交互，完成镜像、容器的管理。
  
![architecture.svg](/img/in-post/post-docker-arch/architecture.svg)

## Docker Engine

![engine-components-flow.png](/img/in-post/post-docker-arch/engine-components-flow.png)

### ~1.11

在Docker 1.11之前，也就是2013/2014~2016早期，Docker Engine的所有功能都包含在一个二进制文件/usr/bin/docker中，它涵盖了：

- Docker Client
- Docker Deamon
- Build tool
- Registry Client

这个docker命令有两种模式，Client模式和Daemon模式，当我们Pull/Push镜像的时候，它处于Client模式。当我们用它启动Docker Daemon（docker daemon ...） 的时候它就处于Daemon模式。也就是说，无论是Docker Client还是Docker Daemon，他们用的是同一个二进制文件/usr/bin/docker。这样的做法优劣并存：

- 容易安装，没有依赖
- 容易升级
- 功能耦合，违背SoC（Separation of Concerns）设计准则
- Daemon挂掉 == Container挂掉

另一个问题就是/usr/bin/docker太过臃肿，所以在2015~2016，对架构进行了调整，将部分功能以插件的形式进行拆分，包括Volume、Network和AuthZ的功能。

## 1.11

2016年4月，Docker 1.11.0版本发布，在该版本中，Docker被拆分成了四部分：

- dockerd
- docker-containerd
- docker-containerd-shim
- docker-runc

> **IMPORTANT**: With Docker 1.11, a Linux docker installation is now made of 4 binaries (`docker`, [`docker-containerd`](https://github.com/docker/containerd), [`docker-containerd-shim`](https://github.com/docker/containerd) and [`docker-runc`](https://github.com/opencontainers/runc)). If you have scripts relying on docker being a single static binaries, please make sure to update them. Interaction with the daemon stay the same otherwise, the usage of the other binaries should be transparent. A Windows docker installation remains a single binary, `docker.exe`.  [1.11.0 release](https://github.com/moby/moby/releases/tag/v1.11.0)

![arch-new.jpeg](/img/in-post/post-docker-arch/arch-new.jpeg)

### Dockerd

docker daemon本身，从Docker 1.12起，docker daemon和docker client被拆分成了两个不同的二进制文件：/usr/bin/docker，/usr/bin/dockerd。

### runC

runC的功能在于运行和管理容器，它是OCI（Open Container Initiative）制定的开放容器格式标准（OCF, Open Container Format）的一种具体实现。2016年Docker将它贡献给了OCI，并被Containerd集成作为其重要的一个模块。

### Containerd

> containerd is an industry-standard container runtime with an emphasis on simplicity, robustness and portability. It is available as a daemon for Linux and Windows, which can manage the complete container lifecycle of its host system: image transfer and storage, container execution and supervision, low-level storage and network attachments, etc.
containerd is designed to be embedded into a larger system, rather than being used directly by developers or end-users.

![containerd-arch.png](/img/in-post/post-docker-arch/containerd-arch.png)

Containerd通过本地Unix socket提供了一组gRPC API，进行镜像、容器等的管理，这些API相对比较底层，可以方便地进行包装和扩展。Containerd使用`runC`来运行容器。并且提供了CLI （ctr）用于开发和调试。

**Containerd和Docker的关系**

![containerd-docker.png](/img/in-post/post-docker-arch/containerd-docker.png)

Containerd的目标就是重构Docker Engine，将镜像分发、网络管理、存储管理等功能从中提取出来，形成可重用的模块。不但用于Docker中，也能方便得被其他容器编排项目使用。

Docker 1.12使用了containerd 0.2.4，包含了容器执行和进程管理的功能，在之后得版本中，越来越多的功能会通过containerd来完成。

**Containerd和runC的关系**

Linux基金会2015年6月成立了OCI组织，旨在围绕容器格式和运行时制定一个开放的工业化标准。而runC是Docker于2016年贡献出来的基于OCI规范的一个具体实现。而Containerd将OCI/runC进行了集成，用于容器的执行，所有runC是Containerd的一个模块。

**Containerd和Kubernetes等容器编排系统的关系**

之前版本的Kubernetes直接使用Docker来管理容器。现在Kubernetes已经能够使用Containderd作为container runtime。比如从Kubernetes 1.10之后，Kubernetes可以将Containderd 1.1用于生产环境中。

![cri-containerd.png](/img/in-post/post-docker-arch/cri-containerd.png)

![cri-plugin-containerd.png](/img/in-post/post-docker-arch/cri-plugin-containerd.png)

### containerd-shim

Containerd-shim是Containerd和Container之间的一个进程，每个运行的Container都会有一个相应的shim进程，它是容器的父进程。shim进程的存在，是的容器和Containerd进程了解耦，这样，Docker daemon和Containerd可以退出和重启，而不会影响到Container的运行。

_**References**_

- https://containerd.io/
- https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/
