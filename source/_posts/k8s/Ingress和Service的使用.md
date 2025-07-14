---
title: Ingress和Service的使用
toc: true
description: more
tags:
  - k8s
categories:
  - k8s
date: 2025-07-02 16:14:04
---
Ingress 和 Service 都是 Kubernetes 中用于**网络访问和流量管理**的关键组件，但它们在**职责**和**功能层面**上有明确的区别，并且通常**协同工作**而非相互替代。

---

### Service (服务)

**Service 的核心作用是为一组 Pod 提供稳定的网络访问。** 把它想象成应用程序的**内部地址簿和基本的流量分配器**。

* **解决 Pod 的不稳定性：** Pod 是短暂的，它们的 IP 地址会随时变化。Service 提供一个**稳定不变的虚拟 IP 地址和 DNS 名称**，无论背后的 Pod 如何变化，客户端总能通过 Service 找到目标应用。
* **负载均衡：** Service 能够将请求**均匀地分发**到其背后所有健康的 Pod 上，实现基本的负载均衡。
* **多种暴露方式：**
    * **ClusterIP (默认)：** 最常用，只能在**集群内部**访问，用于微服务间通信。
    * **NodePort：** 通过集群**每个节点上的一个静态端口**暴露服务，可以在集群外部通过 `节点IP:NodePort` 访问。
    * **LoadBalancer：** 在支持的云环境中，自动创建**云提供商的外部负载均衡器**，并分配一个外部 IP 地址，将流量转发到 Service。
    * **ExternalName：** 将 Service 映射到外部 DNS 名称。
* **适用场景：** 集群内部服务间通信、非 HTTP/HTTPS 协议的服务、或仅需简单外部暴露且能接受非标准端口的情况。

---

### Ingress (入口)

**Ingress 的核心作用是管理来自集群外部的 HTTP/HTTPS 流量，并提供高级路由功能。** 把它想象成集群的**智能门卫和交通指挥员**。

* **统一入口点：** Ingress 作为集群的**单一外部入口**，可以处理所有进入的 HTTP/HTTPS 请求，从而**节省外部负载均衡器成本**。
* **高级路由规则：** 这是 Ingress 最重要的特性。它能根据 HTTP/HTTPS 请求的**域名 (Host)** 和**URL 路径 (Path)** 等规则，将流量智能地路由到集群内部不同的 Service。例如，`yourdomain.com/api` 路由到 `api-service`，`yourdomain.com/blog` 路由到 `blog-service`。
* **SSL/TLS 终止：** Ingress 可以集中处理 SSL/TLS 证书，并在请求到达后端 Service 之前终止 HTTPS 连接，简化了后端应用的配置。
* **依赖 Ingress Controller：** Ingress 资源本身只定义了路由规则，真正实现这些规则的是在集群中运行的 **Ingress Controller**（例如 Nginx Ingress Controller），它会监听 Ingress 资源并相应地配置反向代理。
* **适用场景：** 需要通过一个外部 IP 暴露多个 HTTP/HTTPS 服务、需要基于域名或路径的高级路由、需要集中管理 SSL/TLS 证书、或需要 A/B 测试、流量重写等更复杂的 HTTP 层功能。

---

### 总结区别与联系

| 特性     | Service                               | Ingress                                           |
| :------- | :------------------------------------ | :------------------------------------------------ |
| **层级** | 传输层 (TCP/UDP)为主，也支持部分应用层 | **应用层 (HTTP/HTTPS)** |
| **职责** | 稳定内部访问，基本外部暴露，负载均衡  | **高级外部 HTTP/HTTPS 路由与访问控制** |
| **作用范围** | **集群内部**为主，也可对外暴露        | **集群外部流量导入**，主要针对 HTTP/HTTPS       |
| **路由规则** | 基于 IP 和端口进行转发                | **基于域名、路径等**高级 HTTP/HTTPS 规则          |
| **SSL/TLS** | 不直接提供 SSL/TLS 终止             | **提供 SSL/TLS 终止** |
| **依赖** | 无额外依赖                            | **必须依赖 Ingress Controller** 才能工作        |
| **成本** | 每个 `LoadBalancer` 通常对应一个外部 LB | **一个外部 LB (Ingress Controller) 可服务多个 Ingress** |

**Service 和 Ingress 并非互相替代，而是协同工作。** Ingress 负责将外部的 HTTP/HTTPS 请求根据规则路由到**正确的 Service**，而 Service 则负责将这些请求负载均衡地转发给**最终的 Pods**。

因此，当你的应用需要从外部通过 HTTP/HTTPS 访问，并且需要灵活的域名/路径路由、SSL/TLS 终止或成本效益时，你应该考虑使用 Ingress。而 Service 始终是 Ingress 的后端，负责将流量传递给具体的 Pods。