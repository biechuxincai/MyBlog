---
title: k8s离线部署方案-二进制部署与容器部署介绍
toc: true
description: 二进制或容器化部署 Kubernetes（手动部署） 和使用 kubeadm + 容器镜像（自动部署）对比
tags:
  - k8s
categories:
  - k8s
date: 2025-06-15 10:46:25
---

通过 **二进制部署 Kubernetes（手动部署）** 和使用 **kubeadm + 容器镜像（自动部署）** 是两种完全不同的方式，适合不同层次和需求的用户。下面我给你系统对比一下它们的核心差异：

---

## 🧩 总体概述对比

| 对比项   | kubeadm + 镜像部署                       | 二进制部署                     |
| ----- | ------------------------------------ | ------------------------- |
| 部署方式  | 自动生成配置，自动拉镜像启动组件                     | 手动下载组件、配置、启动              |
| 安装复杂度 | ✅ 简单                                 | ❌ 复杂（全手工）                 |
| 灵活性   | ❌ 受 kubeadm 限制                       | ✅ 自由控制组件配置                |
| 控制力   | 中等                                   | ✅ 最强（从启动参数到证书）            |
| 学习价值  | 入门级                                  | ✅ 进阶/深入理解                 |
| 容器化   | kubelet/etcd/kube-apiserver 都以容器形式运行 | 可以容器化也可以直接运行在系统上（systemd） |
| 证书管理  | kubeadm 自动生成                         | 你需要手动生成、管理 TLS 证书         |
| 版本升级  | kubeadm 支持一键升级                       | 升级较麻烦（手动替换组件 + 升级 etcd）   |
| 故障排查  | 更依赖 `kubeadm logs`                   | ✅ 可以直接查看进程日志、配置，方便深度排查    |
| 适用人群  | 想快速搭建集群的开发者、测试人员                     | 架构师、DevOps、二次开发、安全研究人员    |

---

## ⚙️ 组件运行机制差异

| 组件                      | kubeadm                       | 二进制部署              |
| ----------------------- | ----------------------------- | ------------------ |
| kube-apiserver          | 容器镜像运行，kubelet 通过 manifest 启动 | systemd 或直接运行二进制   |
| etcd                    | 容器形式运行                        | 直接运行 `etcd` 二进制服务  |
| kube-controller-manager | 镜像+manifest                   | 二进制或 systemd       |
| kubelet                 | 本地进程运行，服务由 kubeadm 配置         | 自行配置 systemd 和启动参数 |
| kube-proxy              | 以 Pod 方式运行                    | 可用二进制运行、容器方式也支持    |

---

## 🔐 证书管理对比

| 内容            | kubeadm             | 二进制部署                     |
| ------------- | ------------------- | ------------------------- |
| 证书生成          | 自动通过 kubeadm 生成     | 手动使用 cfssl 或 openssl 自己生成 |
| 证书更新          | kubeadm 自动续签或通过命令更新 | 自己定期更新，可能需要重启组件           |
| kubeconfig 文件 | 自动创建并分发到正确位置        | 需要你自己生成并手动分发              |

---

## 📦 镜像 vs 二进制包管理

| 项目   | kubeadm                    | 二进制部署                   |
| ---- | -------------------------- | ----------------------- |
| 所需资源 | 镜像仓库（需要网络）                 | 二进制文件，离线也可操作            |
| 可控性  | 镜像更新由上游决定                  | 你可以控制每一个二进制版本           |
| 容器依赖 | 强依赖容器引擎（containerd/docker） | 可选是否容器运行，适合极简系统环境（如嵌入式） |

---

## 📈 使用场景适用性

| 使用目的           | 推荐方式             |
| -------------- | ---------------- |
| 快速搭建测试环境       | kubeadm ✅        |
| CI/CD 自动化部署    | kubeadm ✅        |
| 商业生产环境标准部署     | kubeadm ✅ + 部署工具 |
| 学习理解内部机制、精细控制  | 二进制 ✅✅✅          |
| 离线部署、无外网环境     | 二进制 ✅✅           |
| 企业私有化部署 + 安全合规 | 二进制 ✅✅✅（可审计）     |
| 云平台自动运维场景      | kubeadm + 镜像 ✅   |

---

## 🛠️ 总结一句话理解

> kubeadm 是 Kubernetes 官方提供的“半自动化部署工具”，适合快速部署标准集群，而 **二进制部署** 是“从零搭建一个你能完全控制的 Kubernetes 集群”，非常适合深入学习、定制和特殊环境（离线、边缘）使用。

---

如果你目标是：

* 学习底层原理 ✅ 选二进制部署
* 快速部署生产环境 ✅ 用 kubeadm
* 想要自动化批量安装 ✅ 用 Ansible + kubeadm
* 想做定制平台开发 ✅ 二进制部署 + 深入理解启动参数

## 补充
**二进制部署 Kubernetes 是可以容器化的**，只不过这不是默认流程，而是一种自定义架构的部署方式。下面我来详细解答：

---

## ✅ 结论先行：**二进制部署可以容器化运行**

Kubernetes 的每个核心组件（如 `kube-apiserver`、`kube-controller-manager`、`etcd`、`kube-scheduler`、`kubelet`、`kube-proxy`）本质上都是普通的 **Go 语言编译出的二进制文件**。你可以：

* **手动打包它们成容器镜像**
* **使用 systemd 启动容器运行它们**
* **替代 kubeadm 的容器部署流程**

---

## 🔧 示例：将 kube-apiserver 容器化运行

1. 编写 `Dockerfile`

```Dockerfile
FROM scratch
COPY kube-apiserver /usr/local/bin/kube-apiserver
COPY certs /etc/kubernetes/pki/
CMD ["/usr/local/bin/kube-apiserver", "--config=/etc/kubernetes/config.yaml"]
```

2. 构建镜像

```bash
docker build -t my-k8s/kube-apiserver:custom .
```

3. 启动容器运行组件

```bash
docker run -d --name kube-apiserver \
  --network host \
  -v /etc/kubernetes:/etc/kubernetes \
  my-k8s/kube-apiserver:custom
```

---

## ✅ 常见组件的容器化情况对照

| 组件                 | 官方支持容器化？      | 手动二进制部署是否可容器化 | 推荐方式          |
| ------------------ | ------------- | ------------- | ------------- |
| kube-apiserver     | ✅ 是           | ✅ 可以          | 容器运行或 systemd |
| controller-manager | ✅ 是           | ✅ 可以          | 容器运行或 systemd |
| scheduler          | ✅ 是           | ✅ 可以          | 容器运行或 systemd |
| etcd               | ✅ 是           | ✅ 可以          | 容器运行或独立二进制    |
| kubelet            | ❌ 否（必须是宿主机进程） | ❌ 不建议容器运行     | ❗ 必须在宿主机运行    |
| kube-proxy         | ✅ 是           | ✅ 可以          | 容器运行          |

---

## 🚫 注意：kubelet 无法容器化运行！

> kubelet 是 Kubernetes 集群的“宿主机代理”，**负责管理 Pod 的生命周期**，需要直接访问宿主机的：

* cgroups、容器运行时（containerd/socket）
* 主机网络命名空间
* 挂载目录（如 /var/lib/kubelet）

📌 所以 kubelet 必须部署在宿主机中，不能在容器中运行。

---

## ✅ 容器化部署方式的优势

* ✅ 更易于镜像统一管理与版本控制
* ✅ 适配自动化部署工具（如 Docker Compose、K8s 自托管）
* ✅ 易于迁移/备份
* ✅ 便于 DevOps 接管维护流程

---

## 🧠 应用场景总结

| 场景                       | 是否适合容器化运行        |
| ------------------------ | ---------------- |
| 自定义 K8s 实现 / 发行版         | ✅ 是              |
| 高可用组件部署                  | ✅ 是              |
| 节省维护脚本                   | ✅ 是              |
| kubelet、containerd 等系统核心 | ❌ 否              |
| 极简 Linux 或离线部署           | ✅ 是（自建 registry） |

---

## ✅ 如果你想“容器化部署整个 K8s”，推荐以下思路：

* 构建你自己的控制组件容器镜像（apiserver、scheduler 等）
* 使用 `docker compose` 或 `systemd + docker run` 启动这些容器
* kubelet 保持本地运行（不可容器化）

---

📌 **总结一句话**：

> Kubernetes 的二进制组件天然支持容器化，只要注意 `kubelet` 不能容器运行，其他你都可以自由封装、调度、部署在容器中。

---
