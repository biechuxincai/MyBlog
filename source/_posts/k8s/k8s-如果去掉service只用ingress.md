---
title: k8s_如果去掉service只用ingress
toc: true
description: more
tags:
  - k8s
categories:
  - k8s
date: 2025-12-28 22:56:02
---
这个问题问得**非常好，而且很“架构向”**。结论我先放前面，然后慢慢拆开讲清楚：

> **在 Kubernetes 里：
> 👉 标准 / 正统 / 可维护的做法是：Ingress ≠ Service，Ingress 不能“直接替代” Service
> 👉 但：在“技术上”，在某些 Ingress Controller 实现里，确实可以绕过 Service，直接打到 Pod**

下面我分层、分场景讲，尽量不藏着掖着。

---

## 一、K8s 官方设计视角（最重要）

### 1️⃣ Ingress 的“官方定位”

从 **Kubernetes API 设计**角度看：

```text
Ingress = L7 路由规则（HTTP/HTTPS）
Service = 稳定的 L4/L3 访问入口
Pod = 不稳定的运行实体
```

**Ingress 的 backend 设计是：**

```yaml
backend:
  service:
    name: xxx
    port:
      number: 80
```

👉 **Ingress 官方只认 Service，不认 Pod**

这是**非常刻意的设计**，原因有几个：

---

### 2️⃣ 为什么官方一定要 Service？

#### （1）Pod 是不稳定的

* Pod 会：

  * 重建
  * 重启
  * IP 变化
  * 扩缩容

Ingress Controller **不应该**直接感知 Pod 的生命周期细节
👉 这些事情是 **Service 的职责**

---

#### （2）Service 是 K8s 的“抽象边界”

Service 解决了：

| 问题       | Service 做了什么 |
| -------- | ------------ |
| Pod IP 变 | 统一虚拟 IP      |
| 多 Pod    | 负载均衡         |
| Ready 状态 | 自动剔除不健康 Pod  |
| 协议抽象     | TCP / UDP    |

Ingress 的工作是：

> “我只关心：**你后面是谁提供 HTTP 服务**”

---

#### （3）解耦：这是架构层面的东西

如果你：

* 改 Pod Label
* 改 Deployment
* 改副本数

**Ingress 不需要动**

👉 这是经典的“分层抽象”

---

## 二、那为什么你会觉得「好像可以不用 Service」？

你这个感觉**不是错觉**，是因为👇

---

## 三、现实世界：某些 Ingress Controller 的“非标准玩法”

### 1️⃣ 以 NGINX Ingress 为例

**NGINX Ingress Controller 内部其实是：**

* Watch：

  * Ingress
  * Service
  * Endpoints / EndpointSlice

最终生成的是：

```nginx
upstream xxx {
  server 10.1.2.3:8080;
  server 10.1.2.4:8080;
}
```

👉 **最终 NGINX 访问的是 Pod IP**

Service 在中间“只是个信息源”。

---

### 2️⃣ 是否可以绕过 Service？

#### ✔ 技术上：可以（非标准）

一些方式包括：

#### 方法 A：自定义 CRD（如 NGINX Plus / Traefik）

例如：

* Traefik 的 `IngressRoute`
* NGINX 的 `VirtualServer`

可以直接：

```yaml
spec:
  routes:
  - services:
    - name: pod-backend
      kind: Pod
```

⚠️ **这是 Controller 私有能力，不是 K8s 标准**

---

#### 方法 B：Headless Service（折中方案）

```yaml
kind: Service
spec:
  clusterIP: None
```

效果：

* Service 不做负载均衡
* Ingress 直接拿 Pod IP 列表

👉 **几乎等价于“直接打 Pod”，但仍然合法**

这是**最推荐的“绕过式玩法”**

---

### 3️⃣ 那我能不能完全不建 Service？

如果你问的是：

> “我不创建 Service 这个对象，Ingress 还能不能工作？”

答案是：

> ❌ **标准 Ingress：不行**
> ⚠️ **特定 Controller：可能行，但强耦合 + 高风险**

---

## 四、如果你真的“只用 Ingress + Pod”，会发生什么？

### ❌ 你会失去的能力

| 能力                | 结果                         |
| ----------------- | -------------------------- |
| Pod Ready 检测      | 很容易流量打到未就绪 Pod             |
| 自动剔除异常实例          | 依赖 Controller 自己实现         |
| kube-proxy / ipvs | 完全绕过                       |
| 可移植性              | 换 Ingress Controller 基本要重写 |
| 可维护性              | 运维和同事会骂你                   |

---

### ❌ 实际生产中的坑（真实踩过）

* Pod 重建瞬间 502
* 滚动升级期间流量打到旧 Pod
* 扩容后 Ingress 没感知新 Pod
* Endpoint 缓存不同步

---

## 五、什么时候“看起来可以不用 Service”？

### ✅ 合理但少见的场景

1️⃣ **Service Mesh（如 Istio）**

* Ingress Gateway ≠ K8s Ingress
* Sidecar + Envoy 自己做发现
* Service 角色被弱化（但通常仍存在）

---

2️⃣ **单 Pod + 非生产**

* Demo
* 本地测试
* 学习 K8s 内部原理

👉 不推荐生产

---

## 六、推荐结论（很实在）

### ✅ 最佳实践（强烈建议）

```text
Ingress → Service → Pod
```

哪怕你觉得 Service “多此一举”，它依然是：

* 稳定性
* 可维护性
* 团队协作
* 官方支持

的核心组件。

---

### 🟡 如果你真的想“轻一点”

👉 **Headless Service 是最优解**

你依然遵循 K8s 设计，但：

* 不走 kube-proxy
* 不做 ClusterIP 转发
* Ingress 直接感知 Pod IP

---

## 七、一句人话总结

> **Ingress 是门卫，Service 是前台，Pod 是工位
> 你可以让门卫直接记住每个员工的位置
> 但员工一换工位，门卫就懵了**

