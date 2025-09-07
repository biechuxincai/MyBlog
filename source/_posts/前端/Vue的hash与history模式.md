---
title: Vue的hash与history模式
toc: true
description: more
tags:
  - vue
categories:
  - vue
date: 2025-08-20 21:22:21
---
 **Vue Router 的两种路由模式**：

### 1. `/#/` —— **Hash 模式**

* URL 形式：`http://example.com/#/user/123`
* `#` 后面的内容叫 **hash**，属于 **前端路由标记**，不会被浏览器发送到服务器。

  * 例如：请求发到服务器时，服务器只会收到 `/`，不会收到 `#/user/123`。
* 优点：

  * 兼容性好（IE9 及以下也支持）。
  * 部署简单，不需要后端额外配置。
* 缺点：

  * URL 看起来不够美观。
  * SEO 不友好（搜索引擎一般不会抓取 `#` 后的部分）。

---

### 2. `/` —— **History 模式**

* URL 形式：`http://example.com/user/123`
* 使用 HTML5 的 `history.pushState` 和 `popstate` 实现，没有 `#`。
* 请求时，浏览器会把 `/user/123` 直接发给服务器。

  * 如果服务器没有配置 **重定向到 index.html**，就会出现 **404**。
* 优点：

  * URL 简洁、美观，更接近真实网站结构。
  * 更利于 SEO。
* 缺点：

  * 需要服务器额外配置，保证所有路由都能返回入口页面（通常是 `index.html`）。

---

### 总结对比：

| 特性          | Hash 模式 (`/#/`)         | History 模式 (`/`)       |
| ----------- | ----------------------- | ---------------------- |
| URL 是否带 `#` | ✅ 带 `#` (`/#/user/123`) | ❌ 不带 `#` (`/user/123`) |
| 服务端请求       | 只请求 `/`                 | 请求实际路径 `/user/123`     |
| 部署复杂度       | 简单，无需配置                 | 需要后端支持（URL 重定向）        |
| SEO         | 较差                      | 较好                     |
| 兼容性         | 好（旧浏览器也支持）              | 需要支持 HTML5 的浏览器        |

---

👉 所以如果你访问 `http://example.com/#/user/123` 和 `http://example.com/user/123`：

* 前者，服务器只会收到 `/`，Vue 前端自己解析 `#/user/123`。
* 后者，服务器会收到 `/user/123`，如果没配置后端路由转发，就会 404。



# 很多人用 Vue 的 hash/history 模式时，静态资源路径容易踩坑。我们拆开来说。

---

## 🌐 前提：静态资源加载方式

* **绝对路径**：以 `/` 开头，例如 `/js/app.js`
* **相对路径**：不以 `/` 开头，例如 `./js/app.js` 或 `js/app.js`

浏览器解析规则：

* **绝对路径**：相对于网站根目录（domain root）
* **相对路径**：相对于当前 URL 的 **路径部分**（不包含 `#`）

---

## 1. **Hash 模式下**：

URL 示例：

```
http://example.com/#/user/123
```

* **绝对路径** `/js/app.js`

  * 请求 `http://example.com/js/app.js` ✅
  * 不受 `#/user/123` 影响。

* **相对路径** `js/app.js` 或 `./js/app.js`

  * 请求 `http://example.com/js/app.js` ✅
  * 因为 hash（`#/user/123`）不会参与路径计算，浏览器仍然认为当前路径是 `/`。

👉 **总结：Hash 模式对静态资源几乎没有影响**，用绝对路径或相对路径都不会错。

---

## 2. **History 模式下**：

URL 示例：

```
http://example.com/user/123
```

* **绝对路径** `/js/app.js`

  * 请求 `http://example.com/js/app.js` ✅
  * 永远正确（推荐）。

* **相对路径** `js/app.js` 或 `./js/app.js`

  * 请求 `http://example.com/user/js/app.js` ❌（404）
  * 因为浏览器会把当前路径 `/user/123` 当作基准去拼接。

👉 **总结：History 模式必须用绝对路径或正确设置 `<base href="/">`，否则加载资源会错位。**

---

## 3. **举例对比**

| 路由模式    | 当前 URL                          | 资源引用         | 实际请求地址                              | 是否正确 |
| ------- | ------------------------------- | ------------ | ----------------------------------- | ---- |
| Hash    | `http://example.com/#/user/123` | `/js/app.js` | `http://example.com/js/app.js`      | ✅    |
| Hash    | `http://example.com/#/user/123` | `js/app.js`  | `http://example.com/js/app.js`      | ✅    |
| History | `http://example.com/user/123`   | `/js/app.js` | `http://example.com/js/app.js`      | ✅    |
| History | `http://example.com/user/123`   | `js/app.js`  | `http://example.com/user/js/app.js` | ❌    |

---

## 4. **需要注意的点**

1. **Vue CLI / Vite 打包时默认会把静态资源路径处理成绝对路径**（受 `publicPath` / `base` 配置影响）。

   * Vue CLI: `vue.config.js -> publicPath: '/'`
   * Vite: `vite.config.js -> base: '/'`

2. **History 模式下一定要小心相对路径**，因为它依赖当前 URL 路径。

3. 如果你要部署在 **子路径**（比如 `http://example.com/app/`），要正确设置 `publicPath` / `base`，否则静态资源也会 404。

---

