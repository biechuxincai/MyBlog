---
title: tomcat容器运行失败，超出系统资源
toc: true
description: tomcat容器运行失败,Out of system resources
tags:
  - docker
categories:
  - docker
date: 2025-07-14 16:27:31
---

今天要部署一个服务，因为某些限制只能用tomcat来部署，我顺手写了个DockerFile, 结果启动失败，报错 
```bash
Cannot create VM thread. Out of system resources
```
使用docker inspect 查看容器，发现镜像是基于Ubuntu 24 版本的，docker是20.10.9，网上查到
![alt text](../../images/docker/image.png)
没办法，换了个镜像就好了