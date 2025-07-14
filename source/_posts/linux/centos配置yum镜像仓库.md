---
title: centos配置yum镜像仓库
toc: true
description: 在 CentOS 上配置 YUM 镜像仓库，主要是为了将 YUM 源指向国内的镜像站点，这样可以大大提高软件包下载和更新的速度。
tags:
  - linux
categories:
  - linux
date: 2025-06-29 13:32:00
---
-----

在 CentOS 上配置 YUM 镜像仓库，主要是为了将 YUM 源指向国内的镜像站点，这样可以大大提高软件包下载和更新的速度。下面是详细的配置步骤：

-----

## 1\. 备份原有的 YUM 配置文件

在进行任何修改之前，最好先备份当前的 YUM 配置文件，以防万一出现问题可以恢复。YUM 的配置文件通常位于 `/etc/yum.repos.d/` 目录下。

```bash
sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
```

这会将默认的 `CentOS-Base.repo` 文件重命名为 `.bak` 结尾，作为备份。

-----

## 2\. 下载新的镜像仓库配置文件

许多国内的镜像站都提供了直接下载的 YUM 配置文件，你只需要选择一个你觉得速度最快的即可。常用的有阿里云、清华大学、网易等。

### 以阿里云镜像为例：

如果你是 **CentOS 7**，可以下载阿里云的 YUM 配置文件：

```bash
sudo wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
```

如果你是 **CentOS 8** (CentOS Linux 8 已于 2021 年底停止维护，但如果你还在用，可以尝试以下命令，不过更推荐迁移到 CentOS Stream、AlmaLinux 或 Rocky Linux)：

```bash
# 首先备份所有默认的 repo 文件
sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
sudo mv /etc/yum.repos.d/CentOS-AppStream.repo /etc/yum.repos.d/CentOS-AppStream.repo.bak
sudo mv /etc/yum.repos.d/CentOS-Extras.repo /etc/yum.repos.d/CentOS-Extras.repo.bak

# 下载新的 base repo
sudo wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-8.repo
```

**其他常用镜像站点的链接（根据你的 CentOS 版本选择）：**

  * **清华大学镜像站 (TUNA):** `https://mirrors.tuna.tsinghua.edu.cn/help/centos/`
  * **网易镜像站 (163):** `http://mirrors.163.com/.help/centos.html`

你可以访问这些站点的帮助页面，通常会提供最新的 YUM 配置文件下载链接和配置说明。

-----

## 3\. 生成 YUM 缓存

下载并替换了配置文件后，你需要清理旧的 YUM 缓存并生成新的缓存，这样 YUM 才能识别并使用新的镜像源。

```bash
sudo yum clean all
sudo yum makecache
```

  * `sudo yum clean all`: 清除所有已下载的包和元数据缓存。
  * `sudo yum makecache`: 重新生成元数据缓存，从新的镜像源获取软件包列表。

-----

## 4\. 测试 YUM 仓库配置

配置完成后，你可以尝试运行一个 YUM 命令来测试新的仓库是否生效。例如，更新一下软件包列表：

```bash
sudo yum update -y
```

如果更新过程顺利，并且下载速度很快，那就说明你的 YUM 镜像仓库配置成功了！

-----

## 5\. 额外提示：EPEL 仓库配置

EPEL (Extra Packages for Enterprise Linux) 仓库提供了许多 CentOS 官方仓库中没有的额外软件包。如果你需要安装这些软件包，也建议将 EPEL 仓库的源配置到国内镜像。

1.  **安装 EPEL 仓库：**

      * **CentOS 7:**
        ```bash
        sudo yum install -y epel-release
        ```
      * **CentOS 8:**
        ```bash
        sudo dnf install -y epel-release # CentOS 8 开始默认使用 dnf，但 yum 命令仍可用
        ```

2.  **备份 EPEL 配置文件：**

    ```bash
    sudo mv /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel.repo.bak
    sudo mv /etc/yum.repos.d/epel-testing.repo /etc/yum.repos.d/epel-testing.repo.bak
    ```

3.  **下载阿里云的 EPEL 配置文件：**

    ```bash
    sudo wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo # CentOS 7
    # 或者
    sudo wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-8.repo # CentOS 8
    ```

    同样，其他镜像站也有 EPEL 的配置文件。

4.  **清理并生成 EPEL 缓存：**

    ```bash
    sudo yum clean all
    sudo yum makecache
    ```

-----

通过以上步骤，你就可以轻松地将 CentOS 的 YUM 镜像仓库配置到国内的快速源，从而提升软件管理体验。