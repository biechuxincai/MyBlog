---
title: k8s_cni
toc: true
description:  CNI 是怎么把 K8s、容器运行时（Docker / containerd）和网络插件（Flannel / Cilium）真正串起来的从机制层给你拆清楚。

tags:
  - k8s
categories:
  - k8s
date: 2025-12-28 22:57:13
---
 **CNI 是怎么把 K8s、容器运行时（Docker / containerd）和网络插件（Flannel / Cilium）真正串起来的** 从**机制层**给你拆清楚。

下面这套理解，放在你之后看 kubelet 源码、CNI 插件代码、甚至排查 Pod 网络问题时，都会非常有用。

---

## 一、先一句话定性：CNI 到底是啥？

> **CNI 不是一个网络组件，它是一个「接口规范」**

类似 JDBC、SPI、POSIX：

* **Kubernetes / 容器运行时**：只关心「我要一个网络」
* **Flannel / Cilium / Calico**：各自实现「怎么给你网络」
* **CNI**：规定了**双方怎么说话**

所以：

> 👉 **CNI = 协议 + 插件约定，不是网络本身**

---

## 二、整体链路（这是全景图，先记住）

当你 `kubectl run` 或 `Deployment` 起一个 Pod：

```
kubectl
  ↓
apiserver
  ↓
kubelet
  ↓
CRI（containerd / docker-shim）
  ↓
CNI
  ↓
Flannel / Cilium
  ↓
Linux Network（netns / veth / bridge / tc / eBPF）
```

**关键点：**

* K8s 不直接碰网络
* kubelet 不知道 Flannel/Cilium 是谁
* 一切通过 **CNI 二进制 + JSON 配置**完成

---

## 三、CNI 到底“连”的是哪几方？

### 1️⃣ Kubernetes 这边

K8s **只做一件事**：

> **当 Pod Sandbox 创建时，调用 CNI 插件**

这个调用发生在 **kubelet → CRI**

### 2️⃣ Docker / containerd 这边

* Docker / containerd 负责：

  * 创建容器
  * 创建 **network namespace（netns）**
* 但：

  * **不配置 IP**
  * **不连网络**

👉 它把 **netns 路径**交给 CNI

例如：

```
/proc/12345/ns/net
```

---

## 四、CNI 是怎么被“调用”的？

这是很多人模糊的地方，重点来了。

### 1️⃣ CNI 插件的本质

CNI 插件就是：

> **一堆 Linux 可执行文件**

通常在：

```
/opt/cni/bin/
  flannel
  bridge
  host-local
  loopback
  cilium-cni
```

不是 daemon，不是 service，而是：

> **kubelet 在 Pod 创建时直接 fork/exec 它**

---

### 2️⃣ kubelet 如何知道用哪个插件？

靠 **配置文件**：

```
/etc/cni/net.d/
  10-flannel.conflist
  05-cilium.conflist
```

例如 flannel：

```json
{
  "cniVersion": "0.3.1",
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "type": "bridge",
        "bridge": "cni0",
        "ipMasq": true
      }
    }
  ]
}
```

这里已经很关键了：

* kubelet 不关心 flannel 做什么
* 它只看 `type: flannel`
* 然后去 `/opt/cni/bin/flannel`

---

### 3️⃣ CNI 调用时传什么？

靠 **环境变量 + stdin JSON**

核心环境变量：

```bash
CNI_COMMAND=ADD | DEL
CNI_CONTAINERID=<container-id>
CNI_NETNS=/proc/xxx/ns/net
CNI_IFNAME=eth0
```

stdin 里是上面的 JSON 配置

---

## 五、Flannel 是怎么“接入”这套机制的？

### Flannel 的真实角色（别被名字骗）

Flannel **不是 CNI 插件本体**

它其实是 **两部分**：

1️⃣ flanneld（daemon）
2️⃣ flannel CNI 插件（小程序）

---

### 1️⃣ flanneld 在干嘛？

`flanneld` 是一个 **常驻进程**，负责：

* 向 apiserver / etcd 申请：

  * 节点子网（如 10.244.1.0/24）
* 配置：

  * VXLAN / UDP / host-gw
* 写入：

  * `/run/flannel/subnet.env`

例子：

```
FLANNEL_SUBNET=10.244.1.0/24
FLANNEL_MTU=1450
```

---

### 2️⃣ flannel CNI 插件在干嘛？

当 kubelet 调用 CNI：

```
/opt/cni/bin/flannel
```

它做的事是：

1. 读取 `/run/flannel/subnet.env`
2. 把网络参数 **“转交”给真正干活的插件**

比如：

```json
"delegate": {
  "type": "bridge",
  "bridge": "cni0"
}
```

所以真实流程是：

```
kubelet
  → flannel CNI
     → bridge CNI
        → host-local IPAM
```

📌 **Flannel 本身不连 veth，不配 IP**
📌 它是个“中间协调者”

---

## 六、Cilium 是怎么接入 CNI 的？（完全不同流派）

如果说 Flannel 是 **传统 Linux 网络派**

那 **Cilium = eBPF 派**

---

### 1️⃣ Cilium 架构

Cilium 也有两块：

1️⃣ cilium-agent（daemon）
2️⃣ cilium-cni（二进制）

但是它们 **强耦合**

---

### 2️⃣ Cilium CNI 做了什么？

当 Pod 创建：

```
kubelet → /opt/cni/bin/cilium-cni
```

它会：

1. 调用本地 cilium-agent API
2. 请求：

   * 分配 Pod IP
   * 创建 Endpoint
3. agent 用 eBPF：

   * 挂 tc / XDP
   * 建立 L3/L4/L7 转发、策略

关键差异：

| Flannel       | Cilium      |
| ------------- | ----------- |
| bridge + veth | veth + eBPF |
| iptables      | eBPF        |
| 网络 & 安全分离     | 网络 + 安全一体   |
| L3/L4         | L3~L7       |

---

## 七、CNI 的核心抽象（你必须吃透）

### 1️⃣ CNI 只关心三件事

每次 `ADD`：

1. **把 veth 插进 Pod netns**
2. **给 Pod 分 IP**
3. **能让包出去 & 进来**

至于：

* VXLAN / BGP
* eBPF / iptables
* Overlay / Underlay

👉 **全是插件自己的事**

---

### 2️⃣ CNI 不管 Service、不管 DNS

很多人误解：

* CNI = Pod 网络
* kube-proxy / Cilium = Service 网络
* CoreDNS = DNS

这是三个系统

---

## 八、为什么现在 Docker 被“踢出”K8s 了？

你作为工程师，这个历史原因也值得知道。

本质是：

> **K8s 只认 CRI，不认 Docker API**

但：

* **CNI 始终不变**
* Docker / containerd 都只是 CRI 实现

所以你现在看到的是：

```
kubelet → containerd → CNI
```

而不是：

```
kubelet → docker → libnetwork
```

---

## 九、一句话总结（给你一个脑内模型）

> **CNI 是 K8s 和“真实网络世界”之间唯一的标准接口**

* kubelet：我只会调 CNI
* Docker / containerd：我只负责 netns
* Flannel：我管子网 + overlay
* Cilium：我用 eBPF 重写整个数据面
* Linux：我是真正跑包的地方

---


