---
title: k8s离线部署-配置介绍
toc: true
description: 在离线、无外网环境下部署 Kubernetes（二进制方式），最关键且复杂的步骤之一就是：证书与配置文件的准备与生成。
tags:
  - k8s
categories:
  - k8s
date: 2025-06-15 23:05:28
---
**在离线、无外网环境下部署 Kubernetes（二进制方式）**，最关键且复杂的步骤之一就是：**证书与配置文件的准备与生成**。

这部分工作 **kubeadm 可以自动完成**，但在你二进制部署时，必须自己搞定。这正是考验你对 K8s 机制理解的时候。

---

# ✅ 离线部署中涉及的证书与配置文件

我们分为两大类：

## 一、🔐 TLS 证书（基于 CA）

Kubernetes 各组件之间通过 **TLS 加密通信**，如下组件都需要用到证书：

| 组件                 | 使用的证书                    | 说明                       |
| ------------------ | ------------------------ | ------------------------ |
| kube-apiserver     | 使用 CA 颁发的服务端证书           | 被 controller/kubelet 等访问 |
| kubelet            | 连接 kube-apiserver 的客户端证书 | 每个 node 都有               |
| controller-manager | 客户端证书                    | 访问 apiserver             |
| scheduler          | 客户端证书                    | 访问 apiserver             |
| etcd               | 自己的 CA + etcd 证书         | 自签或复用 K8s CA             |
| kube-proxy         | 客户端证书                    | 用于访问 API                 |
| admin 账号           | kubectl 用来访问集群的证书        | kubeconfig 中引用           |

---

## 二、⚙️ 配置文件（KubeConfig）

用于存储认证信息、集群信息、用户身份：

| 文件                            | 作用                   | 包含证书 |
| ----------------------------- | -------------------- | ---- |
| admin.kubeconfig              | 提供 kubectl 用的管理凭据    | ✅ 有  |
| kubelet.kubeconfig            | 节点与 API 通信凭据         | ✅ 有  |
| controller-manager.kubeconfig | 控制器访问凭据              | ✅ 有  |
| scheduler.kubeconfig          | 调度器访问凭据              | ✅ 有  |
| kube-proxy.kubeconfig         | kube-proxy 访问 API 凭据 | ✅ 有  |

---

## ✅ 离线环境如何生成证书与配置？

你需要使用工具（推荐 [`cfssl`](https://github.com/cloudflare/cfssl)）进行证书签发，或使用 `openssl`。

---

## 🛠️ 推荐使用：cfssl 工具

### 1. 安装 cfssl 工具（在联网机器操作一次）

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssl*
mv cfssl* /usr/local/bin/
```

### 2. 生成 CA 根证书（`ca.pem`）

`ca-config.json`：

```json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "87600h"
      }
    }
  }
}
```

`ca-csr.json`：

```json
{
  "CN": "Kubernetes CA",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "L": "Beijing",
    "O": "Kubernetes",
    "OU": "CA",
    "ST": "Beijing"
  }]
}
```

生成：

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

会得到：

```bash
ca.pem
ca-key.pem
ca.csr
```

---

### 3. 生成组件证书（以 kube-apiserver 为例）

`apiserver-csr.json`：

```json
{
  "CN": "kube-apiserver",
  "hosts": [
    "127.0.0.1",
    "10.0.0.1",             // cluster IP
    "192.168.100.10",       // apiserver IP
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "L": "Beijing",
    "O": "Kubernetes",
    "OU": "Cluster",
    "ST": "Beijing"
  }]
}
```

生成：

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  apiserver-csr.json | cfssljson -bare apiserver
```

---

### 4. 用证书生成 kubeconfig 文件

```bash
# 设置环境变量
KUBERNETES_PUBLIC_ADDRESS=192.168.100.10

kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

---

## ✅ 所有证书建议按目录整理（离线打包）

```
/opt/k8s/
├── bin/                    # 所有二进制文件
├── certs/                  # 所有 .pem 文件
│   ├── ca.pem
│   ├── apiserver.pem
│   └── ...
├── config/                 # 所有 kubeconfig
│   ├── admin.kubeconfig
│   ├── kubelet.kubeconfig
│   └── ...
└── manifests/              # 可选 systemd service 或 YAML
```

---

## ✅ 离线部署建议流程

1. 在有网机器准备好：

   * 所有证书（cfssl生成）
   * 所有二进制文件（下载并测试）
   * 所有 kubeconfig 文件

2. 将文件通过 U 盘或内网传输到所有节点

3. 各节点使用这些固定文件初始化

4. 整个流程无需访问公网，也不依赖镜像仓库

---
