---
title: 证书自签与CA签发
toc: true
description: 我们通过一系列 `openssl` 命令手动模拟一次 TLS 证书生成 + 握手验证流程。整个流程可分为 3 个部分：
tags:
  - 网络
categories:
  - 网络
date: 2025-06-15 11:29:36
---

我们通过一系列 `openssl` 命令**手动模拟一次 TLS 证书生成 + 握手验证流程**。整个流程可分为 3 个部分：

---

## ✅ 一、使用 openssl 创建证书（自签）

我们先不接触 CA，直接自签一个证书，模拟服务器部署：

### 1. 生成私钥

```bash
openssl genrsa -out server.key 2048
```

### 2. 生成证书签名请求（CSR）

```bash
openssl req -new -key server.key -out server.csr
```

> 填写过程中，`Common Name` 一定要填你访问的域名，如 `localhost`、`example.com`

### 3. 使用自签方式生成证书

```bash
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

这样我们就生成了：

* 私钥：`server.key`
* 公钥证书：`server.crt`

---

## ✅ 二、使用 openssl 启动一个模拟的 HTTPS 服务端

使用 `openssl s_server` 模拟 HTTPS 服务端监听 4433 端口：

```bash
openssl s_server -key server.key -cert server.crt -port 4433
```

> 📌 此时服务端会监听 `4433`，等待客户端 TLS 握手。

---

## ✅ 三、使用 openssl 模拟客户端发起握手连接

打开另一个终端，用 `openssl s_client` 模拟浏览器发起连接：

```bash
openssl s_client -connect 127.0.0.1:4433
```

你将看到：

* **ClientHello**
* **ServerHello**
* **证书链**
* **握手过程**
* 验证证书签名（自签可能显示不可信）

输出示例：

```
CONNECTED(00000003)
depth=0 CN = localhost
verify error:num=18:self-signed certificate
---
Certificate chain
 0 s:/CN=localhost
   i:/CN=localhost
---
SSL handshake has read 1329 bytes and written 456 bytes
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
...
```

你还可以输入 `GET / HTTP/1.1` 并回车，再回车，来模拟简单的 HTTP 请求。

---

## ✅ 四、其他可观察项

### 查看证书内容

```bash
openssl x509 -in server.crt -noout -text
```

### 查看握手详细过程（包括加密算法、证书校验）

```bash
openssl s_client -connect 127.0.0.1:4433 -tls1_2 -state -debug
```

---

## 🔐 可扩展：用自建 CA 来签发

如果你想进一步模拟“正规签发”流程：

1. 自建 CA 的私钥和证书（根证书）
2. 用 CA 的私钥签发服务器证书
3. 客户端添加 CA 根证书为受信任即可不再报错



---

我们现在来完整演示一遍**使用自建 CA 签发服务器证书的流程**，这比自签更符合实际生产中使用的“权威认证流程”。

---

# ✅ 一、整体流程概览（模拟 PKI）

```text
1. 创建自建 CA（根证书）  ✅
2. 生成服务器私钥 & CSR   ✅
3. 用 CA 签发服务器证书   ✅
4. 使用 openssl 启动服务端 ✅
5. 客户端验证证书链        ✅
```

---

# 🧰 二、实际操作步骤（Linux / Mac 通用）

## 🔸 1. 创建 CA（自建证书机构）

### 1.1 创建 CA 私钥

```bash
openssl genrsa -out ca.key 2048
```

### 1.2 创建 CA 自签证书（有效期 10 年）

```bash
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt
```

> 会要求你输入 CA 名称信息（Common Name 可以写 `My Root CA`）

---

## 🔸 2. 为服务器生成证书请求（CSR）

### 2.1 生成服务器私钥

```bash
openssl genrsa -out server.key 2048
```

### 2.2 生成服务器证书签名请求（CSR）

```bash
openssl req -new -key server.key -out server.csr
```

> 注意 `Common Name` 要填写实际域名，比如 `localhost` 或 `example.com`

---

## 🔸 3. CA 签发服务器证书

### 3.1 创建一个配置文件（可选，但建议）

创建 `v3.ext` 文件：

```ini
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = example.com
```

### 3.2 使用 CA 签发服务器证书

```bash
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out server.crt -days 825 -sha256 -extfile v3.ext
```

最终生成了：

* 服务器私钥 `server.key`
* 服务器证书 `server.crt`（由 CA 签发）
* CA 根证书 `ca.crt`（模拟浏览器预置）

---

## 🔸 4. 启动 TLS 服务端（模拟服务器）

```bash
openssl s_server -key server.key -cert server.crt -CAfile ca.crt -port 4433
```

---

## 🔸 5. 模拟客户端验证证书链

```bash
openssl s_client -connect 127.0.0.1:4433 -CAfile ca.crt
```

若一切成功，输出中应看到：

```
Verify return code: 0 (ok)
```

---

# 🔍 可选调试命令

### 查看 CA 证书内容

```bash
openssl x509 -in ca.crt -noout -text
```

### 查看服务器证书

```bash
openssl x509 -in server.crt -noout -text
```

---

# 📌 常见用途拓展

* 将 `ca.crt` 添加到本地系统受信任根证书（用于浏览器信任）
* 部署到 Nginx、Apache 实现真正 HTTPS 通信
* 构建内部私有 TLS 网络（开发、内网系统等）

---

# 自签证书与自己搭建CA 在签发有什么区别？

这是一个非常关键的问题，关系到**TLS 证书体系的信任模型**和安全边界。

---

## ✅ 一句话区分：

> **自签证书（Self-signed）**：服务器自己当自己的“CA”，没有任何外部信任来源。
> **自建 CA 签发证书**：模拟一个权威认证机构（CA），实现类似浏览器信任链的结构，多个系统或证书可共享该信任。

---

## 🔍 对比自签 vs 自建 CA 签发证书

| 比较维度     | 自签证书（Self-signed） | 自建 CA 签发证书         |
| -------- | ----------------- | ------------------ |
| 签发者      | 服务器自己             | 独立的 CA 私钥签发        |
| 是否可建立信任链 | ❌ 无法建立            | ✅ 可建立完整的 PKI 信任链   |
| 客户端如何信任  | 必须手动信任该服务器证书      | 客户端只需信任一次 CA 根证书   |
| 适合用途     | 测试、个人使用、临时安全传输    | 内部企业系统、集群服务、大规模部署  |
| 证书分发/管理  | 每台机器都要自己签、手动导入    | 签发多个证书，只需统一信任 CA   |
| 吊销/更新机制  | ❌ 无管理能力           | ✅ 可统一管理、吊销、更新、续期   |
| 证书统一控制   | ❌ 没有              | ✅ 有，作为“权威”可以统一管理策略 |
| 安全性（长期看） | 较低（密钥泄露后不易统一处理）   | 高（集中控制、统一吊销/替换）    |

---

## 📦 举个例子：

### ✅ 自签证书（单兵作战）

你用 `openssl req -x509` 直接生成一个 `server.crt`，它自己签了自己。

* 浏览器访问时会报错：“证书未被信任”
* 只能自己导入该证书才不报错
* 若你部署 10 台服务器，需要分别导入 10 个不同的证书

---

### ✅ 自建 CA 签发证书（建立信任体系）

你先生成一个 `ca.crt` 作为“根证书”（只导入一次到客户端系统），然后给每个服务器签发证书：

* 浏览器或系统只需要信任一次 CA
* 所有由这个 CA 签发的服务器证书都自动被信任
* 如果某个证书泄露，可以吊销并重新签发
* 企业内部系统、微服务集群非常适合这种方式

---

## 📌 重点总结

| 类型       | 信任逻辑        | 适用范围              |
| -------- | ----------- | ----------------- |
| 自签证书     | 仅信任“自己”     | 一次性测试、调试场景        |
| 自建 CA 签发 | 构建“组织级信任体系” | 企业内部部署、大规模 TLS 系统 |

---

## 🧠 额外建议：

* 自签证书不建议用于正式环境（除非你完全控制所有客户端）
* 自建 CA 签发非常适合**内部服务**、**Kubernetes 集群**、**VPN 系统**等场景
* 若是公网上服务，推荐用像 **Let’s Encrypt** 这样真实 CA 免费签发证书

---


