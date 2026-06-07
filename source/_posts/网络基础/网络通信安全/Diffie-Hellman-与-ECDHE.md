---
title: Diffie-Hellman\ 与\ ECDHE
toc: true
description: more
tags:
  - 网络安全
categories:
  - 网络安全
date: 2026-06-07 09:47:33
---
如果你问的是 **Diffie-Hellman（DH）和 ECDHE**，那它们是 TLS/HTTPS 中最重要的密钥交换算法之一。

很多人学 TLS 时有个疑问：

> 浏览器已经拿到了服务器公钥，为什么还需要 ECDHE？

答案是：

> RSA/ECDSA 负责证明"你是谁"，ECDHE 负责生成"本次会话密钥"。

这是两个完全不同的问题。

---

# 先理解一个问题

假设：

```text
客户端 A
服务器 B
```

他们第一次见面。

现在希望得到一个：

```text
共享密钥
```

例如：

```text
K = abc123
```

以后所有数据：

```text
AES(K)
```

加密传输。

---

问题来了：

```text
网络被监听
```

如果直接发送：

```text
abc123
```

攻击者也看到了。

那怎么办？

---

# Diffie-Hellman 的伟大发明

1976年，Whitfield Diffie 和 Martin Hellman 提出：

> 双方不用发送密钥，也能算出同一个密钥。

这在当时是革命性的。

---

# 一个简化例子

真实算法涉及大整数和模运算。

先用颜色来理解。

---

公开信息：

```text
黄色
```

大家都知道。

---

客户端秘密选择：

```text
蓝色
```

服务器秘密选择：

```text
红色
```

---

客户端计算：

```text
黄色 + 蓝色
=
绿色
```

发给服务器。

---

服务器计算：

```text
黄色 + 红色
=
橙色
```

发给客户端。

---

攻击者能看到：

```text
黄色
绿色
橙色
```

但看不到：

```text
蓝色
红色
```

---

客户端：

```text
橙色 + 蓝色
=
最终颜色
```

服务器：

```text
绿色 + 红色
=
最终颜色
```

双方得到同一个结果。

---

攻击者没有：

```text
蓝色
红色
```

算不出来。

---

# 真正的 DH

实际使用数学：

公开参数：

```text
g
p
```

所有人都知道。

---

客户端：

随机生成：

```text
a
```

私密。

计算：

A=g^a\bmod p

发送：

```text
A
```

---

服务器：

随机生成：

```text
b
```

私密。

计算：

B=g^b\bmod p

发送：

```text
B
```

---

客户端计算：

K=B^a\bmod p

---

服务器计算：

K=A^b\bmod p

---

数学上可以证明：

```text
结果完全一样
```

即：

```text
K = g^(ab) mod p
```

双方得到了同一个密钥。

---

# ECDH 又是什么

DH 有个问题：

```text
计算量较大
密钥长度较长
```

例如：

```text
2048 bit
3072 bit
```

才能保证安全。

---

后来出现：

**椭圆曲线密码学（ECC）**

对应的 DH 版本：

**ECDH**

全称：

```text
Elliptic Curve Diffie-Hellman
```

---

特点：

同等安全性下：

```text
RSA 3072位
≈
ECC 256位
```

---

因此：

```text
更快
更省CPU
更省带宽
```

---

# ECDHE 是什么

很多人分不清：

```text
ECDH
ECDHE
```

---

ECDH：

固定密钥。

```text
服务器私钥固定
```

---

ECDHE：

E = Ephemeral

意思：

```text
临时的
一次性的
```

---

每次连接：

服务器都会重新生成：

```text
临时私钥
```

例如：

第一次访问：

```text
a1
b1
```

---

第二次访问：

```text
a2
b2
```

---

第三次：

```text
a3
b3
```

---

完全不同。

---

# 为什么必须临时

这是 TLS1.3 的核心思想：

**前向保密（Forward Secrecy）**

---

假设今天：

```text
你访问银行
```

产生：

```text
Session Key #1
```

---

明天：

```text
Session Key #2
```

---

后天：

```text
Session Key #3
```

---

攻击者把所有流量录下来：

```text
2023
2024
2025
```

全部保存。

---

几年后：

服务器私钥泄露。

---

如果使用老式 RSA 密钥交换：

```text
过去所有流量
全部能解密
```

非常危险。

---

如果使用 ECDHE：

即使拿到服务器长期私钥：

```text
也推不出当时的临时密钥
```

因此：

```text
历史通信仍然安全
```

这就是：

```text
Forward Secrecy
前向保密
```

---

# TLS 1.2 和 TLS 1.3

TLS 1.2：

可能出现：

```text
RSA
DHE
ECDHE
```

多种方式。

---

TLS 1.3：

已经强制要求：

```text
(E)DHE
```

也就是：

```text
ECDHE
```

基本成为事实标准。

---

# 浏览器访问 HTTPS 时实际发生什么

以访问：

Google 为例。

```text
1 浏览器发起连接

2 Google发送证书

3 浏览器验证证书

4 双方执行ECDHE

5 得到Session Key

6 使用AES-GCM通信
```

因此：

```text
证书
    ↓
证明身份

ECDHE
    ↓
协商会话密钥

AES
    ↓
真正传输数据
```

这是现代 HTTPS/TLS 的标准流程。

从工程角度看，你可以把 TLS 理解成：

```text
RSA/ECDSA
      ↓
验证服务器身份

ECDHE
      ↓
生成一次性会话密钥

AES-GCM
      ↓
加密业务数据
```

这也是为什么在 Nginx、Java SSL、Spring Security、OAuth2 等场景里，你会经常看到类似：

```text
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
```

这些套件里面已经默认包含 ECDHE 作为密钥交换机制，而不是过去那种 RSA 直接交换密钥的方式。
