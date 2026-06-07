---
title: https全链路分析
toc: true
description: HTTPS 抓包与证书链分析，是理解 TLS 最好的实践方式之一
tags:
  - 网络
categories:
  - 网络
date: 2026-06-07 09:54:13
---
HTTPS 抓包与证书链分析，是理解 TLS 最好的实践方式之一。

很多人学了半天：

```text
RSA
ECDHE
证书
CA
TLS
```

但是不知道这些东西到底在网络里长什么样。

实际上用 Wireshark 抓一次包，你会发现前面学的所有知识都能对应上。

---

# 一、HTTPS 抓包能看到什么

很多人以为：

```text
HTTPS = 完全看不到
```

实际上不是。

HTTPS 只是：

```text
应用数据加密
```

握手过程很多信息仍然是明文。

例如：

```text
TCP三次握手

TLS握手

证书

域名(SNI)
```

都能看到。

---

# 举例

访问：

```text
https://www.baidu.com
```

抓包。

会看到：

```text
TCP

TLSv1.3

Client Hello

Server Hello

Certificate

Application Data
```

---

# 二、TLS握手长什么样

## TCP连接

首先：

```text
Client
    SYN
Server

Server
    SYN ACK
Client

Client
    ACK
Server
```

完成三次握手。

---

然后开始 TLS。

---

# Client Hello

浏览器发送：

```text
Client Hello
```

里面包含：

```text
TLS版本

随机数

支持算法

支持密码套件

支持曲线
```

例如：

```text
TLS_AES_128_GCM_SHA256

TLS_AES_256_GCM_SHA384

TLS_CHACHA20_POLY1305_SHA256
```

---

Wireshark里面类似：

```text
Client Hello
    Version: TLS1.3

    Cipher Suites:
        TLS_AES_128_GCM_SHA256
        TLS_AES_256_GCM_SHA384
```

---

# Server Hello

服务器回复：

```text
Server Hello
```

表示：

```text
我选择TLS1.3

我选择AES128
```

---

还能看到：

```text
ECDHE公钥
```

---

# 三、证书在哪里

后面会出现：

```text
Certificate
```

展开以后：

```text
Certificate
 ├─ Subject
 ├─ Issuer
 ├─ Public Key
 ├─ Signature
```

---

例如：

```text
Subject:
    *.google.com

Issuer:
    Google Trust Services
```

---

这就是服务器证书。

---

# 四、证书里面到底有什么

很多人以为：

```text
证书 = 公钥
```

其实不是。

证书本质上是：

```text
身份证
```

---

里面包含：

```text
域名

公钥

签发机构

有效期

扩展信息

签名
```

例如：

```text
CN=www.google.com

Public Key

Not Before

Not After

CA Signature
```

---

# 五、证书链是什么

这是面试高频问题。

---

浏览器收到：

```text
google证书
```

为什么相信它？

因为：

```text
CA签发
```

---

例如：

```text
Google证书
    ↓
中间CA

Google Trust Services
    ↓
根CA

GlobalSign Root CA
```

形成：

```text
Leaf
  ↓
Intermediate
  ↓
Root
```

---

这就是：

```text
Certificate Chain
```

证书链。

---

# 六、为什么需要中间CA

如果根CA直接签发：

```text
几亿张证书
```

风险巨大。

---

所以现实是：

```text
Root CA
     ↓
Intermediate CA
     ↓
网站证书
```

---

例如：

```text
Root
    私钥离线保存

Intermediate
    在线签发证书
```

---

即使中间CA泄露：

```text
吊销即可
```

根CA仍然安全。

---

# 七、浏览器怎么验证证书链

收到：

```text
Leaf证书
```

例如：

```text
www.google.com
```

---

发现：

```text
Issuer:

Google Trust Services
```

---

于是验证：

```text
Leaf签名
```

是否由：

```text
Google Trust Services
```

签发。

---

然后继续：

```text
Google Trust Services
```

是谁签发的？

---

发现：

```text
GlobalSign Root
```

---

继续验证。

---

最后：

```text
GlobalSign Root
```

就在浏览器内置信任库里。

---

因此：

```text
整条链可信
```

---

# 八、浏览器内置了什么

Windows：

```text
证书管理器
```

Linux：

```text
ca-certificates
```

Chrome：

```text
Root Store
```

---

里面存放：

数百个根证书。

例如：

* DigiCert
* GlobalSign
* ISRG
* Let's Encrypt

---

浏览器真正信任的是：

```text
根证书公钥
```

而不是网站证书。

---

# 九、数字签名如何验证

假设证书内容：

```text
CN=www.test.com

Public Key=ABC
```

---

CA计算：

```text
Hash(证书内容)
```

得到：

```text
123456
```

---

然后：

```text
CA私钥签名
```

得到：

```text
Signature
```

---

浏览器收到：

```text
Certificate
+
Signature
```

---

利用：

```text
CA公钥
```

验证。

---

成功说明：

```text
证书未被修改
```

并且：

```text
确实由该CA签发
```

---

# 十、中间人攻击怎么发生

假设没有证书验证。

---

攻击者：

```text
Client
   ↓
 Hacker
   ↓
 Server
```

---

攻击者返回：

```text
假的公钥
```

客户端也接受。

---

于是：

```text
客户端
↔
攻击者

攻击者
↔
服务器
```

建立两条连接。

---

攻击者可以：

```text
查看

修改

转发
```

所有数据。

---

这就是：

```text
MITM
```

中间人攻击。

---

# 十一、为什么 Fiddler/Charles 能抓 HTTPS

很多开发者第一次用：

* Fiddler
* Charles
* Burp Suite

会很惊讶：

```text
HTTPS不是加密的吗？
```

为什么还能看见请求？

---

因为这些工具会：

```text
自己生成一个根证书
```

例如：

```text
Charles Root CA
```

---

然后安装到系统信任库。

---

浏览器认为：

```text
Charles Root CA
```

可信。

---

访问：

```text
https://www.google.com
```

时：

Charles动态生成：

```text
www.google.com
```

证书。

---

浏览器验证：

```text
签发者 = Charles Root CA
```

发现可信。

---

于是：

```text
浏览器
 ↔ Charles

Charles
 ↔ Google
```

建立两条 TLS。

---

Charles 就能看到：

```text
Request

Response

Cookie

Header

Body
```

全部内容。

---

# 十二、为什么手机抓包经常失败

因为很多 App 开启：

```text
Certificate Pinning
```

证书绑定。

---

例如 App 内写死：

```text
Google证书公钥
```

或者：

```text
证书Hash
```

---

即使系统信任：

```text
Charles Root CA
```

App也会检查：

```text
公钥不匹配
```

然后：

```text
TLS握手失败
```

---

# 十三、开发人员最应该掌握的几个命令

查看证书：

```bash
openssl s_client -connect www.google.com:443
```

查看证书链：

```bash
openssl s_client -showcerts \
-connect www.google.com:443
```

查看详细内容：

```bash
openssl x509 -text -noout
```

---
