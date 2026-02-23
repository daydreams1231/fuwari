---
title: Linux容器化技术
published: 2025-11-26
description: ''
image: ''
tags: [Linux, Docker]
category: ''
draft: false 
lang: ''
---

主流容器引擎有Docker Podman LXC(Linux Container)

Podman可以看作Docker的轻量版本, 无守护进程, rootless, 兼容Docker

Docker和podman一般都是用来部署特定应用的, 部署后开箱即用, 当然你把它当成一个系统用也没什么问题(比如使用容器环境编译软件)

LXC主要用来部署系统, 更适合需要完整操作系统环境的场景. 例如在Armbian上部署一个Ubuntu