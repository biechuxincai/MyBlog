---
title: iptables详解
toc: true
description: more
tags:
  - linux
categories:
  - linux
date: 2025-06-23 23:41:23
---
参考链接
https://lixiangyun.gitbook.io/iptables_doc_zh_cn
http://www.cnblogs.com/metoy/p/4320813.html
http://www.ha97.com/4093.html

`iptables`、`firewalld` 和 **云安全组（Security Groups）** 是网络安全相关的不同层次的防火机制，它们分别在不同的系统或平台中起作用，下面是它们的定义和关系：

---

### 🔸 一、iptables 是什么？

`iptables` 是 Linux 系统中最底层的 **防火墙工具**，用于配置内核中的 `netfilter` 模块。

* 属于 **主机级别的防火墙**。
* 基于规则（rules）控制网络流量的 **过滤、转发、NAT（地址转换）等**。
* 手动操作比较复杂，规则顺序严格。

🧠 举例：

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

表示允许所有进入本机的 22 端口（SSH）TCP 流量。

---

### 🔸 二、firewalld 是什么？

`firewalld` 是 Linux（如 CentOS 7+）引入的一个 **iptables 的前端管理工具**。

* 更方便地管理 iptables 规则。
* 支持 **区域（zone）和服务（service）** 的概念。
* 动态加载规则，不需要重启服务。

🧠 举例：

```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

**👉 实质上 firewalld 是用来间接操作 iptables（或 nftables） 的。**

---

### 🔸 三、安全组（Security Group）是什么？

**安全组是云平台（如阿里云、AWS、腾讯云等）提供的网络访问控制机制**，作用在虚拟机实例（ECS、EC2）之上。

* 是一种 **虚拟防火墙**，作用于 **云平台层面**。
* 控制哪些 IP、端口、协议可以访问你的实例。
* 属于 **平台级别的网络访问控制**。

🧠 举例（阿里云）：

* 放通某 ECS 实例的 80 端口给 0.0.0.0/0（任何公网IP）。
* 禁止其他 IP 访问 22 端口（SSH）。

---

### 🔄 它们之间的关系和区别？

| 比较项   | iptables   | firewalld            | 云安全组（Security Group） |
| ----- | ---------- | -------------------- | -------------------- |
| 所属层级  | OS 内核层     | OS 用户空间工具            | 云平台管理层（如 AWS、阿里云）    |
| 控制范围  | 主机网络流量     | 主机网络流量（通过 iptables）  | 云主机的入站/出站流量          |
| 配置方式  | 命令行手动编写规则  | 使用 zone + service 管理 | Web 控制台或 API         |
| 生效对象  | 本机         | 本机                   | 云主机实例（ECS、VM）        |
| 是否必须有 | 是（Linux默认） | 可选                   | 云平台必须设置              |

---

### ✅ 总结一句话：

* **iptables 是最底层的流量控制工具**；
* **firewalld 是管理 iptables 的工具**；
* **安全组是云平台为实例提供的“外围防火墙”，类似虚拟边界墙**。



`iptables` 中的\*\*自定义链（User-defined chain）\*\*是一个非常实用的功能，能让你将规则模块化、逻辑清晰、便于维护与复用，类似于代码里的“函数”或“子程序”。

---

## 🧠 一、什么是自定义链？

iptables 的每个表（如 `filter`、`nat`）中都包含一些**内置链（如 INPUT、FORWARD、OUTPUT）**，你可以在这些链中定义规则。

**自定义链**是你自己定义的新的规则链，可以在已有链中“跳转”调用，就像函数一样。

---

## 🔧 二、如何创建和使用自定义链？

### ✅ 1. 创建自定义链

```bash
iptables -N MY_CHAIN
```

这会创建一个名为 `MY_CHAIN` 的新链。

---

### ✅ 2. 添加规则到自定义链中

```bash
iptables -A MY_CHAIN -s 192.168.1.100 -j DROP
iptables -A MY_CHAIN -s 192.168.1.200 -j ACCEPT
```

现在这个链的逻辑是：

* 来自 192.168.1.100 的流量被丢弃；
* 来自 192.168.1.200 的流量被接受。

---

### ✅ 3. 在主链中“调用”自定义链

比如你想让 INPUT 流量先走你定义的规则：

```bash
iptables -A INPUT -p tcp --dport 22 -j MY_CHAIN
```

意思是：

* 所有目的端口为 22 的 TCP 流量 → 交给 `MY_CHAIN` 处理。

如果 `MY_CHAIN` 中的规则不匹配，控制流会返回到调用它的 INPUT 链继续往下走。

---

### ✅ 4. 删除自定义链（先清空再删）

```bash
iptables -F MY_CHAIN      # 清空链
iptables -D INPUT -j MY_CHAIN  # 从主链移除引用
iptables -X MY_CHAIN      # 删除链
```

---

## 🎯 示例：阻止某些 IP SSH 登录

```bash
iptables -N BLOCK_BAD_IPS
iptables -A BLOCK_BAD_IPS -s 10.0.0.1 -j DROP
iptables -A BLOCK_BAD_IPS -s 10.0.0.2 -j DROP

iptables -A INPUT -p tcp --dport 22 -j BLOCK_BAD_IPS
```

逻辑是：

1. 目标端口为 22 的 SSH 连接 → 先检查是否是坏 IP；
2. 是坏 IP 则丢弃，否则继续后续规则。

---

## 💡 优势总结

| 优点   | 说明           |
| ---- | ------------ |
| 结构清晰 | 将复杂规则分组、模块化  |
| 易维护  | 修改子链，不影响主链逻辑 |
| 可复用  | 多个主链可调用同一个子链 |
| 降低冗余 | 避免重复写类似规则    |

---

## 📌 注意事项

* 自定义链不能设置为默认链（不能像 INPUT 那样自动接收包）。
* `iptables -L` 不会显示调用路径，要用 `iptables-save` 或 `iptables -v -L` 看得更清楚。
* 链之间的 `-j` 跳转默认是“有状态”的，会在子链执行完后回到主链继续往下匹配。

---

Docker 在 Linux 系统中会**自动配置 iptables 规则**，用来实现容器网络的隔离、端口映射、NAT 转换等功能，完全不需要你手动设置。但了解它的配置方式，对调试、防火墙策略设置、安全加固都非常关键。

下面我来系统讲解：

---

## 🧠 一、Docker 默认如何使用 iptables？

### ✅ Docker 启动时：

1. **创建 bridge 网络（默认是 `docker0`）**
2. **自动设置 iptables NAT 规则**
3. **创建以下 iptables 链（在 `filter` 和 `nat` 表）**：

常见的自动链有：

| 表        | 链             | 功能                        |
| -------- | ------------- | ------------------------- |
| `nat`    | `DOCKER`      | 实现容器端口的 DNAT 转换           |
| `filter` | `DOCKER`      | 控制访问容器的包是否允许              |
| `filter` | `DOCKER-USER` | 用户自定义链，**优先于 DOCKER 链执行** |
| `filter` | `FORWARD`     | 控制容器之间的流量转发               |

---

## 🔍 二、实际效果示例

假设你运行了：

```bash
docker run -d -p 8080:80 nginx
```

Docker 会做以下事情：

### 1️⃣ `nat` 表中添加 DNAT 规则（端口映射）

```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

### 2️⃣ `filter` 表中允许转发数据包

```bash
iptables -A FORWARD -d 172.17.0.2/32 -p tcp -m tcp --dport 80 -j ACCEPT
```

### 3️⃣ `POSTROUTING` 做 SNAT 转换（容器访问外部）

```bash
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

---

## 🚧 三、和 firewalld 的关系

如前所述，**Docker 直接操作 iptables，不通过 firewalld**，所以你在 `firewall-cmd` 中看不到 Docker 映射的端口。

为避免冲突或失效，建议：

```bash
firewall-cmd --permanent --zone=trusted --add-interface=docker0
firewall-cmd --reload
```

---

## 🛠 四、配置 docker 是否使用 iptables

Docker 默认开启 iptables 支持：

```bash
dockerd --iptables=true
```

但你可以禁用（不推荐）：

```bash
dockerd --iptables=false
```

这将禁用所有自动规则，端口映射等功能也会失效，你需要**手动配置所有 iptables 规则**。

---

## 🧱 五、可插入自定义规则的链：DOCKER-USER

Docker 会自动跳过 `INPUT` 和 `OUTPUT`，所有跨容器/外部访问的流量最终走 `FORWARD`，你可以使用 `DOCKER-USER` 链添加自定义规则：

```bash
iptables -I DOCKER-USER -s 1.2.3.4 -j DROP
```

这样你可以禁止某个 IP 访问所有容器，而不影响 Docker 自动生成的规则。

---

## 🧪 六、查看当前 docker 的 iptables 配置

```bash
iptables -t nat -L -n -v
iptables -t filter -L -n -v
```

重点关注 `DOCKER`、`DOCKER-USER`、`FORWARD` 链等。

---

## ✅ 总结一张图理解 Docker 网络与 iptables 关系

```
         [客户端请求: 8080]
                |
          PREROUTING (nat)
                |
        +---> DOCKER (DNAT 到容器IP:80)
        |
        v
     FORWARD (filter)
        |
        +---> DOCKER / DOCKER-USER (检查、放行)
        |
        v
  容器 eth0（172.17.0.X:80）
```

---

