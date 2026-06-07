---
title: cilium运行原理简介
toc: true
description: more
tags:
  - k8s
categories:
  - k8s
date: 2025-12-28 23:30:56
---
这个问题问得已经不是「怎么用 Cilium」，而是**在问它“到底重写了 Linux 网络的哪一层、用了什么模型”**。
我会直接给你一个**工程师视角、可落地、可对源码和 bpftool 对照的解释**，不走营销路线。

你可以把这篇当成：**Cilium 的“内核级网络架构说明书”**。

---

# 一、先给结论（非常重要）

> **Cilium 不是一个 CNI + 网络插件**
> **它是：一个基于 eBPF 的 L3–L7 分布式网络与安全数据面**

换句话说：

* Flannel / Calico：**配置 Linux**
* Cilium：**在 Linux 里“写代码”**

这是本质差异。

---

# 二、Cilium 的整体网络模型（一句话）

> **“Endpoint-centric + eBPF direct routing”**

拆开来就是三件事：

1. **Endpoint 是一等公民（不是 IP、不是网段）**
2. **数据包在 veth 入口就被 eBPF 接管**
3. **转发 / NAT / LB / Policy 都在 eBPF 里完成**

---

# 三、Cilium 的网络拓扑模型（不是 bridge / overlay）

## 1️⃣ Flannel / Calico 的经典模型

```text
Pod
 ↓
veth
 ↓
bridge (cni0)
 ↓
iptables / route
 ↓
vxlan
```

这是：

> **Node-centric + 网段模型**

---

## 2️⃣ Cilium 的模型（重点）

```text
Pod
 ↓
veth
 ↓
eBPF (tc ingress)
 ↓
eBPF (L3/L4/L7 decision)
 ↓
direct redirect
 ↓
veth / vxlan / eth
```

这是：

> **Endpoint-centric + Flow-based**

---

# 四、Endpoint-centric 是什么鬼？

在 Cilium 里：

* **每个 Pod = 一个 Endpoint**
* 每个 Endpoint：

  * 有 ID（不是 IP）
  * 有 label
  * 有 policy
  * 有 BPF map

你可以看到：

```bash
cilium endpoint list
```

输出里有：

```text
ENDPOINT ID
IP
LABELS
BPF MAP
```

📌 **IP 只是 endpoint 的一个属性，不是核心**

---

# 五、Cilium 的真实数据路径（内核级）

下面是你真正要理解的。

---

## 1️⃣ veth ingress：eBPF 第一刀

在 Pod → Node 的 veth ingress：

```text
tc ingress
  └── bpf_lxc.o
```

它做什么？

* 标记来源 endpoint ID
* 做 NetworkPolicy 判断
* 决定：

  * drop
  * redirect
  * 继续

⚠️ **这里还没进 Linux IP 栈**

---

## 2️⃣ L3 路由逻辑（不是路由表）

eBPF 查的是：

```text
endpoint_map:
  10.244.2.20 → endpoint_id=5678 → node=192.168.1.11
```

而不是：

```bash
ip route
```

---

## 3️⃣ Service / LB 在哪？

### kube-proxy 模型（传统）

```text
iptables PREROUTING
  DNAT → Pod IP
```

### Cilium 模型（核心差异）

```text
tc ingress
  └── bpf_lb.o
       └── rewrite dst IP / port
```

📌 **conntrack 不是必需**
📌 L4 LB 完全在 BPF

---

## 4️⃣ 跨 Node 转发（两种模式）

### 模式 A：VXLAN（overlay）

```text
eBPF
  → vxlan0
    → eth0
```

### 模式 B：Native Routing（推荐）

```text
eBPF
  → eth0
```

前提：

* 云厂商 VPC
* BGP
* 同一 L2

📌 这就是 Cilium 性能好的根本原因之一

---

## 5️⃣ Node2：直接命中目标 Endpoint

在 Node2：

```text
tc ingress
  └── bpf_host.o
       └── redirect to vethXXX
```

不经过：

* bridge
* iptables
* kube-proxy

---

# 六、Cilium 的网络模型分层（这是“心法”）

你可以把 Cilium 看成 5 层：

```
┌─────────────────────────┐
│ L7: HTTP/gRPC (Envoy)   │
├─────────────────────────┤
│ L4: TCP/UDP (eBPF LB)   │
├─────────────────────────┤
│ L3: IP routing (eBPF)   │
├─────────────────────────┤
│ L2: veth / vxlan / eth  │
├─────────────────────────┤
│ Linux kernel (eBPF)     │
└─────────────────────────┘
```

📌 Flannel / Calico 只在 L2–L3

---

# 七、Cilium 是怎么“替代”Linux 网络组件的？

| Linux 组件   | Cilium 怎么干掉它  |
| ---------- | ------------- |
| bridge     | eBPF redirect |
| iptables   | eBPF program  |
| conntrack  | BPF map       |
| kube-proxy | eBPF LB       |
| tc rules   | eBPF          |

---

# 八、Cilium 的“代价”（不粉饰）

你说“不隐瞒”，那就讲真话。

### 优点

* 高性能
* 可观测性强（flow / Hubble）
* L7 policy
* 云原生友好

### 缺点

* 内核要求高（>= 5.10 推荐）
* 排错门槛高
* 出问题你得懂 eBPF
* 比 Flannel 更“黑魔法”

---

# 九、一个工程师级结论

> **Cilium 的网络模型不是 Overlay vs Underlay**
> **而是：把“网络决策”从用户态和 iptables，提前到内核入口**

它本质上是：

> **一个在 Linux 内核里运行的分布式虚拟交换机 + 防火墙 + 负载均衡器**

---

## 如果你愿意继续，我可以往更“硬”的方向带你：

* 一个 TCP SYN 包在 bpf_lxc 里的完整执行路径
* Cilium 的 BPF map 设计（endpoint / service / conn）
* Cilium 如何实现 L7（为什么要 Envoy）
* 为什么 tcpdump 抓不到 Service 流量
* Cilium 和 Calico eBPF 的根本差异

你点一个，我继续。
这个问题问得**非常对路**，已经完全是「看懂 Cilium 源码之前必须搞明白的那一步」。

你问的其实不是“路由”，而是：

> **Cilium 是怎么在完全不依赖 Linux 路由表的情况下，知道：
> 👉 这个 Pod 在哪、该往哪个 Node 转发、该 redirect 到哪个 veth？**

我会按**真实运行机制**来讲，不抽象、不玄学。

---

# 一、先给一句话答案（先定锚）

> **Cilium“学习”Pod 路由的方式不是 sniff、不是算出来的，而是：
> 👉 通过 K8s 控制面 + 自己的 Agent 主动“构建一张全局 Endpoint 路由表”，并下发到内核 eBPF map。**

📌 **不是数据平面学习**
📌 **是控制平面同步**

---

# 二、Cilium 的两张世界地图（核心认知）

你必须把 Cilium 分成两个世界：

```
┌───────────────┐
│ 控制平面      │  ← cilium-agent（用户态）
└───────────────┘
         ↓ 同步
┌───────────────┐
│ 数据平面      │  ← eBPF（内核态）
└───────────────┘
```

**“学习路由”发生在控制平面，不在内核里。**

---

# 三、第一步：Pod 创建时发生了什么？

我们从最源头开始。

---

## 1️⃣ kubelet 创建 Pod

* containerd 创建 netns
* CNI 调用 `cilium-cni`

---

## 2️⃣ cilium-cni 干了啥？

它不是“配网络”那么简单，而是：

1. 请求 `cilium-agent`：

   ```text
   POST /endpoint
   ```
2. 把关键信息交上去：

   * Pod IP
   * Pod labels
   * netns path
   * container ID
   * Node 名字

📌 **这一步是 Cilium 认知世界的起点**

---

## 3️⃣ cilium-agent 创建 Endpoint 对象

Agent 内部生成一个：

```text
Endpoint {
  ID: 1234
  IP: 10.244.1.10
  Node: node1
  Labels: app=web
}
```

这个 Endpoint 同时存在于：

* agent 内存
* etcd / k8s CRD（视配置）
* BPF map（稍后）

---

# 四、第二步：Cilium 如何“知道别的 Node 上的 Pod”？

重点来了。

---

## 1️⃣ Cilium 监听 K8s API（Watcher）

`cilium-agent` 会 watch：

* Pods
* Nodes
* Services
* Endpoints
* CiliumNode CRD

例如：

```bash
kubectl get ciliumnodes
```

你会看到：

```yaml
spec:
  addresses:
    - ip: 192.168.1.11
  podCIDRs:
    - 10.244.2.0/24
```

---

## 2️⃣ 每个 Node 都会发布“我负责哪些 Pod IP”

Node2 启动时：

* 申请 PodCIDR
* 创建 `CiliumNode` 对象
* 向集群广播：

```text
10.244.2.0/24 → via 192.168.1.11
```

📌 这一步 ≈ **分布式路由通告**
📌 但不是 BGP（除非你开了）

---

# 五、第三步：Agent 构建“全局路由视图”

在 Node1 上：

`cilium-agent` 最终得到：

```text
Local endpoints:
  10.244.1.10 → endpoint_id=1234

Remote nodes:
  10.244.2.0/24 → node_ip=192.168.1.11
```

然后做关键动作：

> **把这些信息写进 eBPF map**

---

# 六、核心数据结构：BPF Map（这是真相）

这是 eBPF 能“路由”的全部原因。

---

## 1️⃣ Endpoint Map（本地 Pod）

```text
Key: Pod IP
Value:
  endpoint_id
  veth_ifindex
  security_identity
```

查到就 **直接 redirect 到 veth**

---

## 2️⃣ Node CIDR Map（远端 Pod）

```text
Key: 10.244.2.0/24
Value:
  node_ip=192.168.1.11
  encap=VXLAN
```

查到就 **封装 → 发给 Node2**

---

## 3️⃣ Service Map（顺带）

```text
Key: 10.96.0.1:80
Value:
  backend pod list
```

---

📌 **内核数据面不“学习”，只“查表”**

---

# 七、数据包来了之后，内核怎么“用”这些表？

真实路径是这样的：

---

## Pod A 发包

```
10.244.1.10 → 10.244.2.20
```

---

## eBPF 程序执行逻辑（伪代码）

```c
if (lookup(endpoint_map, dst_ip)) {
    redirect_to_veth();
} else if (lookup(node_cidr_map, dst_ip)) {
    encap_and_forward();
} else {
    pass_to_kernel();
}
```

📌 没有路由协议
📌 没有 ARP
📌 没有广播

---

# 八、那如果 Pod / Node 变了呢？

### Pod 删除

* K8s 事件 → agent
* agent 删除 Endpoint
* 更新 BPF map

### Pod 漂移 / 重建

* 新 Endpoint
* 新 IP
* 原 map 删除，插入新项

### Node Down

* CiliumNode 消失
* 所有节点删除对应 CIDR
* 流量立刻失败（fail fast）

---

# 九、这和传统“路由学习”的本质差异

| 传统网络         | Cilium    |
| ------------ | --------- |
| ARP / MAC 学习 | 控制平面下发    |
| OSPF / BGP   | K8s Watch |
| 路由表          | BPF map   |
| 动态学习         | 事件驱动      |

一句话：

> **Cilium 的“路由学习”本质是：
> 用 Kubernetes 代替了传统网络的控制协议。**

---

# 十、终极工程师级总结

> **Cilium 并不是“聪明到自己学会路由”**
> **而是：**
>
> * 用 K8s API 当控制平面
> * 用 cilium-agent 构建全局视图
> * 用 eBPF map 把视图固化进内核
> * 用 eBPF 程序在数据面查表转发

这是为什么它：

* 快
* 稳
* 但复杂

---

## 如果你愿意下一步继续“下钻”：

* 一个 endpoint 创建到 BPF map 的完整调用栈
* BPF map 的 key/value 结构设计
* Cilium 用不用 ARP？为什么？
* 开启 BGP 后路由是怎么传播的？
* Cilium 和 Service Mesh 的关系

你点一个，我继续给你拆到源码级别。
