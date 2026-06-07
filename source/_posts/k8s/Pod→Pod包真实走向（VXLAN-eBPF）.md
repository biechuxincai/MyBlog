---
title: Pod→Pod包真实走向（VXLAN/eBPF）
toc: true
description: more
tags:
  - k8s
categories:
  - k8s
date: 2025-12-28 22:58:23
---
好，这个点非常值得**彻底吃透**。
我按你的要求：**不回避细节、不美化实现、按真实数据路径讲**，而且我会把 **Flannel(VXLAN)** 和 **Cilium(eBPF)** 放在**同一坐标系里对照**，这样你一眼就能看出「旧世界」和「新世界」的根本差异。

你可以把下面内容当成一次**“网络包的尸检报告”**。

---

# 一、统一前提（先把变量锁死）

假设场景：

* Pod A

  * IP：`10.244.1.10`
  * Node1：`192.168.1.10`
* Pod B

  * IP：`10.244.2.20`
  * Node2：`192.168.1.11`
* Pod A → Pod B
* 跨 Node

---

# 二、Flannel（VXLAN）真实包路径（传统派）

## 1️⃣ Pod A 发包的那一刻

**Pod 内部：**

```text
App
 ↓
TCP/UDP
 ↓
eth0 (veth peer)
```

此时：

```
src = 10.244.1.10
dst = 10.244.2.20
```

---

## 2️⃣ veth：Pod → Node

```
Pod netns
  eth0
    │
    └── vethXXX ── vethYYY
                     │
                 Node netns
```

包从 vethYYY 进入 **宿主机网络栈**

---

## 3️⃣ Linux 路由判定（关键点）

Node1 路由表里有：

```bash
10.244.0.0/16 dev flannel.1
```

说明：

> Pod 网段的包 → 走 flannel VXLAN 接口

---

## 4️⃣ VXLAN 封装（第一次“套娃”）

原始 Pod 包被**整体包起来**：

```text
[ Outer Ethernet ]
[ Outer IP   src=192.168.1.10 dst=192.168.1.11 ]
[ UDP 4789 ]
[ VXLAN Header (VNI) ]
---------------------------------
[ Inner IP src=10.244.1.10 dst=10.244.2.20 ]
[ TCP/UDP ]
[ Payload ]
```

📌 这一步是 **内核 VXLAN 模块**做的
📌 flanneld 只负责提前配置好规则

---

## 5️⃣ 物理网络传输

这已经是一个**普通 UDP 包**

* 交换机 / 路由器：

  * 完全不知道 Pod 的存在
  * 只看 192.168.1.10 → 192.168.1.11

---

## 6️⃣ Node2 解封装

到 Node2 后：

```text
UDP 4789
 ↓
VXLAN 模块
 ↓
剥掉外层头
 ↓
恢复 Pod 原始 IP 包
```

---

## 7️⃣ 进入 cni0 bridge

```text
flannel.1
  ↓
cni0 (Linux bridge)
  ↓
vethYYY
  ↓
Pod B eth0
```

Pod B 收到包：

```
src=10.244.1.10
dst=10.244.2.20
```

**成功**

---

## 8️⃣ Flannel VXLAN 总结（说人话）

> Flannel 的思路：
> **“我在 Node 层搞一个虚拟二层网络，把 Pod 全部塞进去”**

代价：

* 每个包都：

  * 封装 + 解封装
  * 走 iptables / bridge
* 性能一般
* 排错极其痛苦（tcpdump 看一堆套娃）

---

# 三、Cilium（eBPF）真实包路径（新派）

现在来真的硬货。

---

## 1️⃣ Pod A 发包

一样：

```
src=10.244.1.10
dst=10.244.2.20
```

进入 veth

---

## 2️⃣ veth ingress：eBPF 提前拦截

⚠️ **这里是第一个本质差异**

在 veth 入口：

```
tc ingress
  └── eBPF program
```

eBPF 在这里做：

* 校验安全策略（NetworkPolicy）
* 查 endpoint map
* 判断目标 Pod 在哪

📌 **还没进 Linux IP 栈**

---

## 3️⃣ Cilium 判断：跨 Node

eBPF 查到：

```text
10.244.2.20 → Node2 → 192.168.1.11
```

---

## 4️⃣ Cilium 封装（但不是传统 bridge）

### 情况 A：VXLAN 模式（最常见）

封装类似，但 **由 eBPF 控制路径**

```text
Outer IP src=192.168.1.10 dst=192.168.1.11
UDP 8472 (Cilium VXLAN)
Inner IP 그대로
```

⚠️ **没有走 bridge / iptables**

---

### 情况 B：Native Routing（无封装）

如果你是：

* 同一二层
* 或 BGP / 云厂商 VPC

那么：

> **完全不封装**

直接：

```
src=10.244.1.10
dst=10.244.2.20
```

物理网络直达

（这是 Flannel 做不到的）

---

## 5️⃣ Node2：eBPF 直接投递 Pod

在 Node2：

* VXLAN 解封装
* eBPF 程序直接：

```
redirect → veth → Pod B
```

📌 不进 bridge
📌 不查 iptables
📌 不靠 conntrack

---

## 6️⃣ Cilium Service 的“顺手一刀”

如果目标是 Service IP：

```
10.96.0.1:80
```

eBPF 会：

* 在 ingress 阶段
* 直接改写为某个 Pod IP

> kube-proxy 直接失业

---

## 7️⃣ Cilium 总结（说实话）

> **Cilium 的核心思想：**
>
> “Linux 网络栈太慢，我绕过去”

优势：

* 少一次内核路径
* 少一堆规则
* 可做到 L7（HTTP / gRPC）

代价：

* 实现复杂
* Debug 难（你得会 bpftool）
* 内核要求高

---

# 四、两条路径的“照妖镜级对比”

| 维度            | Flannel VXLAN | Cilium eBPF |
| ------------- | ------------- | ----------- |
| 网络模型          | 虚拟二层          | 直路由         |
| 封装            | 必须            | 可选          |
| iptables      | 重度依赖          | 几乎不用        |
| bridge        | 有             | 无           |
| Service       | kube-proxy    | eBPF        |
| NetworkPolicy | iptables      | eBPF        |
| 性能            | 一般            | 高           |
| 排错            | tcpdump       | bpftool     |

---

# 五、你在排错时能“看到”的差异

### Flannel：

```bash
ip link
  flannel.1
  cni0

iptables -t nat -L
```

### Cilium：

```bash
bpftool prog
bpftool map
cilium bpf endpoint list
```

---

# 六、一句终极理解（非常重要）

> **Flannel 是“用 Linux 网络拼出来的集群网络”**
> **Cilium 是“在 Linux 内核里写了一个新网络”**

不是营销语，是真相。

---
