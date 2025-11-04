---
title: APK介绍
toc: true
description: more
tags:
  - android
categories:
  - android
date: 2025-11-01 14:59:54
---
## AndroidManifest.xml

应用清单文件，版本号、版本名称、权限、应用名称、应用图标、活动、服务、广播等配置信息，均在这个文件中。

## classes.dex

安卓代码文件，在开发安卓应用时，开发者编写的 Java 代码会先编译成 N 个 class 文件，最后再将它们合并成 classes.dex 文件，所以应用 Java 层的代码基本都在该文件中，有些比较大的应用还有 classes2.dex、classes3.dex…

## resources.arsc

资源配置文件，包含着字符串、主题、图片文件索引、布局文件索引等信息。

## res/

资源文件夹，包含着图片、布局、菜单等文件，resources.arsc 中索引的文件均存放在该目录中（进行过资源混淆的除外）。

## assets/

副资源文件夹，不是每个 APK 中都有它，里面的文件也不会在 resources.arsc 中进行索引。

## lib/

存放 so 文件，该类文件通常由 C / C++ 进行编写，MT 目前不支持修改 so 文件。

## META-INF/

存放签名相关的文件，通常没有什么用。
