---
title: centos查看系统硬件信息
toc: true
description: more
tags:
  - linux
categories:
  - linux
date: 2025-06-17 12:49:29
---

 在 CentOS 中，可以使用以下方法查看系统的硬件信息，包括 CPU、内存、磁盘、网络和其他组件。

1. 查看 CPU 信息
```bash
cat /proc/cpuinfo
查看核心数量：
grep -c ^processor /proc/cpuinfo
查看具体型号：
grep "model name" /proc/cpuinfo | uniq
```
2. 查看内存信息
```bash
cat /proc/meminfo
查看总内存大小：
free -h
```
3. 查看磁盘信息
```bash
查看磁盘分区和使用情况：
df -h
查看磁盘设备：
lsblk
查看磁盘详细信息：
fdisk -l
```
4. 查看网络信息
```bash
查看网络接口和 IP 地址：
ip addr
查看网络配置：
ifconfig
查看活动网络连接：
netstat -tuln
```
5. 查看主板和 BIOS 信息
```bash
查看主板信息：
dmidecode -t baseboard
查看 BIOS 信息：
dmidecode -t bios
```
6. 查看 PCI 设备
```bash
lspci
如果 lspci 未安装：
sudo yum install pciutils -y
```
7. 查看 USB 设备
```bash
lsusb
如果 lsusb 未安装：
sudo yum install usbutils -y
```
8. 查看系统信息总结
```bash
使用 uname 查看内核和架构：
uname -a
使用 hostnamectl 查看主机和操作系统信息：
hostnamectl
```
9. 使用 inxi 查看详细硬件信息
```bash
inxi 是一个强大的工具，用于查看系统硬件和软件的详细信息。

安装 inxi
sudo yum install epel-release -y
sudo yum install inxi -y
使用 inxi
查看所有硬件信息：
inxi -F
查看 CPU、内存、磁盘等：
inxi -C -m -D
```
10. 使用 lshw 查看硬件信息
```bash
lshw 是另一个工具，用于生成硬件的详细报告。

安装 lshw
sudo yum install lshw -y
使用 lshw
查看所有硬件信息：
sudo lshw
查看特定硬件：
sudo lshw -short
```
11. 其他工具
```bash
硬盘 SMART 信息：
sudo yum install smartmontools -y
sudo smartctl -a /dev/sda
查看 RAID 信息：
sudo yum install mdadm -y
sudo mdadm --detail /dev/md0
```