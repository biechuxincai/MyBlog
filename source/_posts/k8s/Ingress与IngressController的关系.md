---
title: Ingress与IngressController的关系
toc: true
description: Ingress与IngressController的关系
tags:
  - null
categories:
  - null
date: 2025-08-05 09:19:59
---

---

## ✅ 一、Ingress 与 Ingress Controller 的关系

| 概念                          | 说明                                                                                    |
| --------------------------- | ------------------------------------------------------------------------------------- |
| **Ingress（资源对象）**           | 是 Kubernetes 的一种 API 资源，用来**定义外部请求如何路由到集群内部的服务**（类似一个规则）。                             |
| **Ingress Controller（控制器）** | 是一个实际运行的程序，比如 `nginx-ingress-controller`，它会**监听 Ingress 资源，并配置反向代理（如 NGINX）来实现访问规则**。 |

🧠 你可以理解为：

> `Ingress` 是“规则”，
> `Ingress Controller` 是“执行规则的人”。

---

## ✅ 二、部署 nginx-ingress controller（最常见控制器）

### ✅ 方法一：用 Helm 安装（推荐）

如果你的集群能联网，直接执行：

```bash
# 添加官方 Helm 仓库
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 安装 ingress-nginx 控制器
helm install my-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

这个会：

* 在 `ingress-nginx` 命名空间中安装 ingress controller
* 自动创建 Deployment、Service、RBAC、ConfigMap 等资源

> 安装后会启动一个 NGINX Pod，在每次你创建 Ingress 资源时自动应用规则。

---

### ✅ 方法二：使用 YAML 方式（不需要 Helm）

执行命令：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

> 如果你是裸金属（物理机），请用这个版本：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
```

---

## ✅ 三、部署好后如何使用 Ingress

以下是一个完整示例：

### 1. 创建一个服务（例如 nginx）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

### 2. 创建一个 Ingress 资源

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-nginx
                port:
                  number: 80
```

### 3. 访问方式

* 你需要将 `myapp.example.com` 解析到 ingress controller 的 IP（NodePort、LoadBalancer、或 host 网络）
* 比如：

  ```bash
  curl -H "Host: myapp.example.com" http://<IngressControllerIP>
  ```

---

## ✅ 总结（一句话记住）

* `Ingress` 定义规则：访问哪个域名/路径，去哪个 Service。
* `Ingress Controller` 是控制器，执行这些规则（如 nginx-ingress）。
* 你必须**先安装 Ingress Controller**，然后再创建 Ingress 资源，才能生效。

---

如果你是 **离线环境**、**裸金属服务器**，或需要 **NodePort 替代 LoadBalancer** 的方式部署
