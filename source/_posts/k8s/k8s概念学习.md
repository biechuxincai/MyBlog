---
title: k8s概念学习
toc: true
description: 学习 Kubernetes（k8s）各个组件的概念是理解其集群架构和工作原理的基础。
tags:
  - k8s
  - 容器
  - 运维
categories:
  - k8s
date: 2025-06-14 17:39:47
---
学习 Kubernetes（k8s）各个组件的概念是理解其集群架构和工作原理的基础。下面我将按照控制平面（Control Plane）和节点组件（Node Components）两个大类，对常见组件进行概念介绍，并附带常见用途和图示建议结构，方便你理解整体架构。

---

## ✅ 一、Kubernetes 架构总览

一个 K8s 集群包含两大部分：

* **控制平面（Control Plane）**：负责集群的管理和调度。
* **工作节点（Worker Node）**：运行 Pod（容器的载体）。

```
 控制平面（Master）
 ├─ kube-apiserver
 ├─ etcd
 ├─ kube-scheduler
 └─ kube-controller-manager
       └─ cloud-controller-manager（可选）

 工作节点（Node）
 ├─ kubelet
 ├─ kube-proxy
 └─ container runtime（如 containerd、Docker、CRI-O）
```

---

## ✅ 二、控制平面组件

| 组件                               | 作用                        | 常见用途                             |
| -------------------------------- | ------------------------- | -------------------------------- |
| **kube-apiserver**               | 集群 API 接口入口，是 K8s 的“中枢神经” | 所有命令、组件间通信都通过它发起                 |
| **etcd**                         | 分布式 KV 存储，用于保存集群状态        | 保存所有资源状态（Pod、Service、ConfigMap等） |
| **kube-scheduler**               | 负责 Pod 调度到合适节点            | 判断哪个 Node 最适合运行新的 Pod            |
| **kube-controller-manager**      | 负责控制器逻辑，如副本控制、节点管理        | Deployment控制、自动扩容、重建、状态保持等       |
| **cloud-controller-manager**（可选） | 与云平台对接，管理云资源              | 例如 AWS/GCP 的负载均衡器、磁盘等资源          |

---

## ✅ 三、节点组件

| 组件             | 作用                           | 常见用途                     |
| -------------- | ---------------------------- | ------------------------ |
| **kubelet**    | 每个 Node 上的核心守护进程，控制 Pod 生命周期 | 保证容器健康、状态上报、执行指令         |
| **kube-proxy** | 网络代理，维护 Node 网络规则            | 实现 Pod 的服务发现和负载均衡        |
| **容器运行时**      | 运行容器的底层工具                    | containerd、Docker、CRI-O等 |

---

## ✅ 四、扩展性组件（可选/增强）

| 组件                 | 作用                       |
| ------------------ | ------------------------ |
| CoreDNS            | DNS 服务，实现 Service 发现     |
| Metrics Server     | 提供资源监控数据（CPU、内存）         |
| Ingress Controller | 控制 HTTP/HTTPS 入口流量       |
| CSI 插件             | 容器存储接口（提供挂载磁盘）           |
| CNI 插件             | 容器网络接口（如 Calico、Flannel） |

---

## ✅ 五、简化理解图示（建议）

你可以画出如下图结构：

```
                    控制平面
  ┌─────────────────────────────────────────────┐
  │ kube-apiserver ─── etcd                    │
  │        │                                   │
  │        ├─ kube-scheduler                   │
  │        └─ kube-controller-manager          │
  └─────────────────────────────────────────────┘
                │
    ┌───────────┼──────────────────────────────────────────┐
    ▼           ▼                                           ▼
 ┌───────┐   ┌───────┐         ...                      ┌───────┐
 │ Node1 │   │ Node2 │                                 │ NodeN │
 └───────┘   └───────┘                                 └───────┘
    │           │                                          │
 kubelet     kubelet                                  kubelet
 kube-proxy  kube-proxy                               kube-proxy
 containerd  containerd                               containerd
    ▲           ▲                                          ▲
    └────运行 Pod（容器）────┘            ─────运行业务应用─────┘
```

---

