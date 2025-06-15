---
title: TLS（SSL）握手的全过程
toc: true
description: 完整演示一次 TLS（SSL）握手的全过程
tags:
  - 网络
  - 协议
categories:
  - 网络
date: 2025-06-15 11:23:26
---
我们来**完整演示一次 TLS（SSL）握手的全过程**，以 **TLS 1.2/1.3 常见流程**为核心，分阶段说明每一步发生了什么，涉及了哪些技术原理和数据交换。

---

## 🧩 场景设定

你在浏览器访问：[https://example.com](https://example.com)
客户端：浏览器
服务端：部署了 TLS 证书的 Web 服务器（如 Nginx）

我们以 **TLS 1.2** 为主讲解，因为其握手更复杂、可理解性强，后面再补充 TLS 1.3 的优化。

---

## 🔐 TLS 1.2 握手过程详解（客户端和服务端）

```
    客户端                          服务端
      |   ---- ClientHello -------->   |
      |                                |
      |   <---- ServerHello --------   |
      |   <---- Certificate --------   |
      |   <---- ServerHelloDone ----  |
      |                                |
      | -- PreMasterSecret加密发送 --> |
      |                                |
      | <-- 服务器计算共享密钥 ------- |
      | -- ChangeCipherSpec ---------> |
      | -- Finished -----------------> |
      |   <---- ChangeCipherSpec ---- |
      |   <---- Finished ------------ |
```

---

### 🔸 第一步：ClientHello

浏览器发出 `ClientHello` 请求，内容包括：

* 支持的 TLS 版本（如 TLS 1.2）
* 支持的加密算法（Cipher Suites）
* 客户端生成的随机数 `client_random`
* 支持的压缩算法
* 可选：支持的扩展（如 SNI、ALPN）

---

### 🔸 第二步：ServerHello + Certificate

服务器响应：

* 选定 TLS 协议版本和加密算法
* 服务器生成的随机数 `server_random`
* 服务器的 **数字证书**（包含域名、公钥、签名等）
* ServerHelloDone 表示“我说完了”

**此时客户端已经获取到服务端的公钥**。

---

### 🔸 第三步：密钥协商（PreMaster Secret）

客户端：

* 生成一个随机的 **PreMasterSecret**
* 使用服务器的 **公钥加密 PreMasterSecret**
* 把密文发送给服务器（只有服务器才能解密）

服务器：

* 用自己的私钥解密得到 PreMasterSecret

双方：

* 利用：

  ```
  master_secret = PRF(pre_master_secret, client_random + server_random)
  ```

  生成对称密钥，之后通信使用对称加密（如 AES）

---

### 🔸 第四步：完成握手

1. 客户端发送：

   * `ChangeCipherSpec`：我之后使用加密通信了
   * `Finished`：我计算的握手摘要（用对称加密加密）

2. 服务器也发送：

   * `ChangeCipherSpec`
   * `Finished`

💡 若双方“Finished”校验一致，说明握手成功，接下来就是加密数据通信。

---

## 📦 握手完成后，数据通信开始

之后的数据都使用 **对称加密算法（AES、ChaCha20）**，用协商出的密钥加密，并加上 HMAC 校验防篡改。

---

## ⚡️ TLS 1.3 的变化（对比优化）

TLS 1.3 取消了很多冗余流程，握手更快、更安全：

| 项目     | TLS 1.2   | TLS 1.3            |
| ------ | --------- | ------------------ |
| 握手轮次   | 多轮        | 最多 1 RTT（0-RTT 支持） |
| 加密算法   | 可选 RSA/DH | 只支持前向安全（ECDHE）     |
| 加密开始时间 | 握手后       | ServerHello 之后就加密  |
| 证书传输   | 明文        | 加密                 |

---

## 🧪 补充：如何抓包观察 TLS 握手？

用 Wireshark：

1. 打开浏览器访问 https 网站
2. 抓取 TCP 443 端口的数据包
3. 搜索关键词：`ClientHello`、`ServerHello`、`Certificate`、`Finished`

你可以清楚看到整个握手过程和交换的随机数、证书链等。

---

## 🧠 总结

TLS 握手的核心是：

1. **建立身份认证（服务器证书）**
2. **协商出对称加密的密钥（通过非对称）**
3. **后续通信都加密，并验证完整性**

---
