---
title: matchLabels
toc: true
description: k8s 中matchLabels  的多个kv 必须都满足吗
tags:
  - k8s
categories:
  - k8s
date: 2025-08-05 09:10:34
---
\*\*在 Kubernetes 中，`matchLabels` 中的多个 `key: value` 条件必须 **全部满足**（即 **逻辑与 AND**）才能匹配资源。

---

### 举个例子：

```yaml
selector:
  matchLabels:
    app: nginx
    tier: frontend
```

这个 selector 会选中具有以下标签的 Pod 或其他资源：

```yaml
metadata:
  labels:
    app: nginx
    tier: frontend
```

如果某个 Pod 缺少其中任何一个标签（比如只有 `app: nginx` 没有 `tier: frontend`），就**不会被选中**。

---

### 小结：

* `matchLabels` 是一个 **key-value map**，**所有的键值对都必须匹配**（逻辑 **AND**）。
* 想要更复杂的匹配逻辑（如 OR、NOT），可以使用 `matchExpressions`。

---

如需实现更灵活的选择条件，可以这样写：

```yaml
selector:
  matchExpressions:
    - key: app
      operator: In
      values: ["nginx", "httpd"]
    - key: tier
      operator: NotIn
      values: ["backend"]
```

这个例子表示：

* `app` 是 `nginx` 或 `httpd`
* 并且 `tier` 不是 `backend`

---

