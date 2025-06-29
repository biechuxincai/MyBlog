---
title: git配置大小写敏感
toc: true
description: 拉去代码后发现远程的文件名字跟本地大小写不一样
tags:
  - Git
categories:
  - Git
date: 2025-06-26 14:42:08
---

拉去代码后发现远程的文件名字跟本地大小写不一样


```bash
git config core.ignorecase # 查看当前是否识别大小写

```
若为 true 则配置为false
```bash
git config core.ignorecase  false # 查看当前是否识别大小写

```