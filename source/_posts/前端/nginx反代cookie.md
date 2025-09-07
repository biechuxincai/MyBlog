---
title: nginx反代cookie
toc: true
description: more
tags:
  - nginx
  - 反代
categories:
  - nginx
date: 2025-08-22 06:11:44
---
## proxy_cookie_domain配置误区

　　Nginx做反向代理的时候，我们一般习惯添加proxy_cookie_domain配置，来做cookie的域名转换，比如

```yaml

location /api {
   proxy_pass https://***.test.com;
   proxy_cookie_domain b.test.com  a.test.com;
}

```
　　最近在项目中发现，不配置这个属性，依然运转正常，我发现自己一直以来可能理解错了这个选项。我们首先来看下proxy_cookie_domain的官方定义：
```yaml

Syntax: proxy_cookie_domain off;
proxy_cookie_domain domain replacement;
Default:
proxy_cookie_domain off;
Context: http, server, location
This directive appeared in version 1.1.15.

Sets a text that should be changed in the domain attribute of the “Set-Cookie” header fields of a proxied server response. Suppose a proxied server returned the “Set-Cookie” header field with the attribute “domain=localhost”. The directive proxy_cookie_domain localhost example.org will rewrite this attribute to “domain=example.org”.
```
　　大致意思：proxy_cookie_domain参数的作用是转换response的set-cookie header中的domain选项，由后端设置的域名domain转换成你的域名replacement，来保证cookie的顺利传递并写入到当前页面中，注意proxy_cookie_domain负责的只是处理response set-cookie头中的domain属性，仅此而已。

## 在涉及第三方认证场景下的反代

CAS 这种第三方认证场景就比普通 API 复杂了 ⚡，因为 **CAS 登录过程中会涉及跨域重定向 + Cookie 设置**，如果只是简单的反代，登录往往会失败。我们可以按步骤拆解下：

------

## 🔑 CAS 登录流程（简化版）

1. 用户访问业务系统 `https://app.a.com`
2. 系统发现未登录 → 跳转 CAS `https://auth.third.com/cas/login?service=https://app.a.com/cas/callback`
3. 用户在 `auth.third.com` 登录 → CAS 返回 `ticket` 并重定向到 `https://app.a.com/cas/callback?ticket=xxx`
4. 系统用 `ticket` 去 `auth.third.com` 校验，拿到用户信息，建立本地会话。

------

## ❌ 问题点（直接反代会遇到的坑）

1. **跨域 Cookie 不生效**
   - `auth.third.com` 登录后设置的 `Set-Cookie: JSESSIONID=xxx; Domain=auth.third.com`
   - 浏览器只会保存到 `auth.third.com`，业务域 `app.a.com` 用不到。
2. **重定向域名冲突**
   - 如果直接反代 `auth.third.com` → `sso.a.com`，CAS 返回的 `Location: https://auth.third.com/cas/login?...` 还是原域名，浏览器绕过了反代。

------

## ✅ 解决思路

要想 **把第三方 CAS 登录完全融入自己的域名**，反代需要处理：

1. **域名统一**：通过 Nginx 把 `auth.third.com` 代理到 `sso.a.com`，所有请求都走自己域名。
2. **重写 Cookie**：把 `Set-Cookie: Domain=auth.third.com` 改成 `sso.a.com` 或 `.a.com`。
3. **重写 Location**：CAS 的重定向 `Location: https://auth.third.com/...` 改成 `https://sso.a.com/...`。

------

## 🔧 Nginx 配置示例

假设：

- CAS 原始域名：`auth.third.com`
- 我们对外暴露：`sso.a.com`

```nginx
server {
    listen 443 ssl;
    server_name sso.a.com;

    ssl_certificate /etc/nginx/certs/a.com.pem;
    ssl_certificate_key /etc/nginx/certs/a.com.key;

    location / {
        proxy_pass https://auth.third.com/;
        proxy_set_header Host auth.third.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 修改返回头的 Set-Cookie 域名
        proxy_cookie_domain auth.third.com sso.a.com;

        # 修改重定向地址中的域名
        proxy_redirect https://auth.third.com/ https://sso.a.com/;
    }
}
```

------

## 🔍 认证流程变为

1. 用户访问 `https://app.a.com`
2. 系统跳转 `https://sso.a.com/cas/login?...`
3. 实际被 Nginx 转发到 `https://auth.third.com/cas/login?...`
4. 用户在 `sso.a.com` 下完成 CAS 登录，Cookie 被重写为 `Domain=sso.a.com`，能被浏览器保存。
5. CAS 登录成功后返回 `Location: https://sso.a.com/cas/callback?ticket=xxx`（重写后的地址）。
6. 浏览器回到 `app.a.com` 完成校验。

------

## ⚠️ 注意事项

- **服务 URL 一致性**：CAS 校验时 `service` 参数必须和配置一致，比如 `https://app.a.com/cas/callback`，否则验证会失败。

- **跨域 Cookie 限制**：如果 `app.a.com` 需要同时访问 `sso.a.com` 的 cookie，建议把 cookie 域名改成 `.a.com`。

  ```nginx
  proxy_cookie_domain auth.third.com .a.com;
  ```

- **前端 AJAX 调用**：如果业务系统要直接调 `sso.a.com`，记得配 `Access-Control-Allow-Credentials` + `withCredentials:true`。

------

👉 所以总结：
 第三方 CAS 认证的反代核心就是 **“域名伪装 + Cookie 域名重写 + 重定向地址重写”**。简而言之就是反代一切！

------

