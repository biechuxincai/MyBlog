---
title: k8s集群服务配置域名
toc: true
description: 一套 **从零开始的完整 Ingress-Nginx（NodePort 模式）安装 + 域名配置 + 常见问题排查** 的完整教程。内容偏实战、踩坑规避，全流程能直接在你现在的 K8s 集群使用。
tags:
  - null
categories:
  - null
date: 2025-08-12 22:28:08
---
一套 **从零开始的完整 Ingress-Nginx（NodePort 模式）安装 + 域名配置 + 常见问题排查** 的完整教程。内容偏实战、踩坑规避，全流程能直接在你现在的 K8s 集群使用。

---

# 🚀 0. 前置条件说明（确保你环境符合）

* 已安装 Kubernetes（kubeadm / k3s / kind 都可以）
* 集群能正常部署 Pod、Service
* 每个节点已经配置 CNI（比如 Flannel、Calico）
* 将通过 **NodePort 暴露 Ingress Controller**

NodePort 是最适合你当前环境（无云 LB、无域名）的方式。

---

# 🧩 1. 安装 ingress-nginx（NodePort 版本）

我们直接使用官方 YAML，不需要 helm。

### **1.1 下载官方部署文件**

官方稳定版（推荐）：

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.0/deploy/static/provider/baremetal/deploy.yaml
```

⛔ 注意：
默认这是 **NodePort 模式**（适用于物理机/自建集群）。

---

# 🧩 2. 修改 ingress-nginx 让它使用固定 NodePort（可选但强烈推荐）

默认部署后端口随机，你可能想固定为：

* HTTP：30080
* HTTPS：30443

执行：

```
kubectl -n ingress-nginx edit svc ingress-nginx-controller
```

修改成：

```yaml
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 443
    nodePort: 30443
```

保存退出。

---

# 🧩 3. 检查 Ingress Controller 是否正常运行

```
kubectl get pod -n ingress-nginx -o wide
```

你应该看到：

```
ingress-nginx-controller-xxxx   1/1   Running   node ip...
```

服务暴露情况：

```
kubectl get svc -n ingress-nginx
```

结果类似：

```
NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)
ingress-nginx-controller   NodePort   10.0.123.45      <none>        80:30080/TCP,443:30443/TCP
```

---

# 🧩 4. 域名指向 NodePort（重要）

你需要将你的域名（比如 api.example.com）解析到集群内任意一个 node IP，如：

```
api.example.com -> 192.168.1.20
```

如果你有多个节点：

```
api.example.com -> 每个节点 IP（建议多条 A 记录）
```

访问方式：

```
http://api.example.com:30080/
```

⚠️ 如果你希望不带端口访问 → 要用反向代理或公网 LB，这属于后续操作，先把本教程跑通。

---

# 🧩 5. 创建一个测试应用（Deployment + Service）

创建 test-app.yaml：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: test
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: test-service
spec:
  selector:
    app: test
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

应用：

```
kubectl apply -f test-app.yaml
```

---

# 🧩 6. 创建 Ingress（绑定域名 → 路由服务）

创建 ingress-test.yaml：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
```

应用：

```
kubectl apply -f ingress-test.yaml
```

---

# 🧩 7. 测试访问

浏览器访问：

```
http://api.example.com:30080
```

你应该看到 nginx Welcome 页面。

如果看到 404：

用：

```
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

检查是否域名匹配出错。

---

# 🧩 8. 配置 HTTPS（可选）

如果你没有公网，不想用 Let's Encrypt，可以用自签证书：

```
openssl req -x509 -nodes -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=api.example.com"
```

创建 secret：

```
kubectl create secret tls api-tls --key tls.key --cert tls.crt
```

修改 ingress-test.yaml：

```yaml
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
```

应用：

```
kubectl apply -f ingress-test.yaml
```

访问：

```
https://api.example.com:30443
```

---

# 🧩 9. 常见坑（强烈建议看一遍）

### ❌ DNS 不生效

➡️ 域名没解析到 Node IP
➡️ 或本地 DNS 缓存

### ❌ 404 Not Found

➡️ IngressClassName 必须为 `nginx`
➡️ host 必须和访问的域名一致

### ❌ 返回 default backend

➡️ NodePort 端口没开放（需要关闭防火墙或允许端口转发）

### ❌ HTTPS 证书错误

➡️ 自签证书必须 CN=SNI 域名一致

### ❌ Ingress Controller 处于 Completed 状态

➡️ 不是 controller，是 job（patch / create job）正常现象。

---

# 🧩 10. 最后给你一份“一步到位的完整 YAML 打包”

如果你愿意，我可以给你：

* ingress-nginx NodePort 固定端口版本 YAML
* 测试应用 YAML
* 完整 ingress 示例
* TLS + 非 TLS 双版本


