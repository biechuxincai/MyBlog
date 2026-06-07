---
title: kube-proxy有什么用跟cni插件有什么关系
toc: true
description: more
tags:
  - k8s
categories:
  - k8s
date: 2025-12-28 23:26:18
---
这个问题问得**非常到位，而且已经到 K8s 网络“核心层”了**。
很多人用 K8s 用了很久，其实对 **kube-proxy 和 CNI 的分工是模糊的**，你现在正好踩在这个关键点上。

我先给你一句**结论级总览**，然后我们慢慢拆、拆到实现层。

---

# 一句话总览（先建立正确直觉）

> **CNI 负责“Pod 能不能互相通信”
> kube-proxy 负责“Service 的流量怎么转发到 Pod”**

两者 **不是一类东西，也不互相替代**，但**强烈配合**。

---

# 一、先搞清楚：K8s 网络到底分几层？

我们把 K8s 网络粗暴但准确地分成 3 层：

```
① Pod 网络层（谁能 ping 谁）
② Service 抽象层（虚拟 IP 怎么用）
③ 入口流量层（Ingress / LB）
```

| 层级            | 负责组件              |
| ------------- | ----------------- |
| Pod ↔ Pod     | **CNI 插件**        |
| Service → Pod | **kube-proxy**    |
| HTTP 路由       | Ingress / Gateway |

你问的 **kube-proxy vs CNI** 正好是 **① vs ②**

---

# 二、CNI 插件：它到底“干了什么活”

### 1️⃣ CNI 是什么？

CNI（Container Network Interface）本质上是：

> **“容器网络插件规范 + 插件实现”**

它解决的是最底层的问题：

> 👉 **Pod 启动后，有没有网卡？IP 是啥？能不能通信？**

---

### 2️⃣ CNI 在 Pod 创建时做了什么？

当 kubelet 创建 Pod 时：

```
kubelet
  ↓
调用 CNI 插件
  ↓
插件做：
  - 创建 veth pair
  - 分配 Pod IP
  - 配置路由 / bridge / vxlan / ebpf
```

结果：

```text
Pod = 一个有独立 IP 的“主机”
```

---

### 3️⃣ 不同 CNI 干的事一样吗？

**目标一样，手段完全不同**

| CNI     | 网络模型                 |
| ------- | -------------------- |
| flannel | overlay (vxlan)      |
| calico  | 路由 / iptables / eBPF |
| cilium  | eBPF                 |
| weave   | overlay              |

但它们**都不管 Service**

---

### 4️⃣ CNI 的核心职责（非常重要）

✔ Pod IP 分配
✔ Pod ↔ Pod 通信
✔ Node ↔ Pod 通信

❌ 不负责虚拟 IP
❌ 不负责负载均衡
❌ 不关心 Service 名字

---

# 三、kube-proxy：它存在的“唯一理由”

### 1️⃣ kube-proxy 解决的是什么问题？

K8s 有个东西叫：

```yaml
kind: Service
spec:
  clusterIP: 10.96.0.10
```

问题来了：

> **10.96.0.10 这个 IP 在哪？
> 哪个网卡？谁监听？**

答案是：

> 👉 **没人监听，这是个“虚拟 IP”**

kube-proxy 就是来：

> **“让这个虚拟 IP 看起来像真的一样”**

---

### 2️⃣ kube-proxy 的本质

> kube-proxy = **Service 的流量转发实现器**

它 Watch：

* Service
* Endpoints / EndpointSlice

然后在 Node 上做三件事之一（取决于模式）：

---

### 3️⃣ kube-proxy 的三种模式（重点）

#### 🟡 userspace（几乎淘汰）

* 用户态转发
* 慢
* 早期方案

---

#### 🟢 iptables（最常见）

```text
访问 ClusterIP
  ↓
iptables DNAT
  ↓
Pod IP:Port
```

优点：

* 简单
* 稳定

缺点：

* 规则多了性能下降
* 随机负载均衡

---

#### 🔵 IPVS（性能更好）

* 内核态 LB
* 支持多种调度算法
* 更稳定

---

### 4️⃣ kube-proxy **并不转发 Pod ↔ Pod 流量**

⚠️ 这是很多人误解的点：

```text
Pod A → Pod B
```

👉 **完全走 CNI**

```text
Pod → Service → Pod
```

👉 **kube-proxy 介入**

---

# 四、kube-proxy 和 CNI 的关系（重点总结）

### 🔗 它们是“上下游关系”

```
Pod 发包
  ↓
CNI 保证包能走到 Node / 网络
  ↓
kube-proxy 决定：
  这个包是不是 Service 流量？
  要不要 DNAT？
```

### 可以理解为：

* **CNI = 修路**
* **kube-proxy = 设路牌 + 红绿灯**

---

# 五、如果没有 kube-proxy 会怎样？

### ❌ Service 直接废掉

* ClusterIP 不能访问
* NodePort 不能访问
* LoadBalancer 不能访问

但注意：

👉 **Pod IP 仍然能 ping 通**（CNI 还在）

---

# 六、为什么现在有人“不用 kube-proxy”？

你可能听过：

> “Cilium 可以不用 kube-proxy”

这是 **真的，但有前提**。

---

### 1️⃣ Cilium kube-proxy replacement

Cilium 用 **eBPF** 实现：

* Service LB
* DNAT
* Health Check

👉 **它不是不用 Service**
👉 **它是“自己实现了 kube-proxy 的功能”**

kube-proxy 这个 **进程**没了
但 **“kube-proxy 这件事”还在**

---

### 2️⃣ 本质没变

| 组件         | 干的活                  |
| ---------- | -------------------- |
| kube-proxy | Service 转发           |
| Cilium     | 用 eBPF 重写 Service 转发 |

---

# 七、Ingress、kube-proxy、CNI 放在一起看

完整链路：

```text
Client
  ↓
Ingress Controller (Pod)
  ↓
Service (ClusterIP)
  ↓
kube-proxy (DNAT)
  ↓
Pod IP
  ↓
CNI 网络
```

如果是：

* **Headless Service**：kube-proxy 几乎不参与
* **直连 Pod IP**：kube-proxy 完全不参与

---

# 八、一句不官方但很准的总结

> **CNI 决定“你能不能打到我”
> kube-proxy 决定“你打到我之后，我帮你转给谁”**

---

如果你愿意，下一步我可以帮你：

* 用 **iptables 规则实例**拆 kube-proxy
* 对比 **iptables vs IPVS vs eBPF**
* 结合你现在用的 **CNI（calico / cilium / flannel）**画一条真实流量路径

你这个问题已经明显不是“新手级”了，继续往下挖会很有意思。
