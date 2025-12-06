---
title: so文件解析
toc: true
description: Linux 下的 `.so`（shared object）本质上就是 **ELF（Executable and Linkable Format）格式的动态链接库**。
tags:
  - linux
categories:
  - linux
date: 2025-12-03 22:18:02
---

---

# 🔶 一、`.so` 文件是什么？

Linux 下的 `.so`（shared object）本质上就是 **ELF（Executable and Linkable Format）格式的动态链接库**。

ELF 本身可用在三种场景：

* 可执行程序（ET_EXEC）
* 可重定位目标文件（ET_REL，如 `.o`）
* 动态库（ET_DYN，即 `.so`）

`.so` ≈ 一种特殊的可执行文件类型，只不过需要动态链接器来加载。

---

# 🔶 二、ELF `.so` 文件的结构（宏观）

你可以把 `.so` 文件想象成一个“多段+多表”的结构，核心如下：

```
+--------------------------+
| ELF Header               |
+--------------------------+
| Program Header Table     | 运行时使用（装载）
+--------------------------+
| Section Header Table     | 链接器用（链接时）
+--------------------------+
| .text      程序代码段     |
| .data      初始化数据段   |
| .bss       未初始化数据段 |
| .rodata   常量字符串等    |
| .plt/.got 动态链接辅助段   |
| .dynamic   动态段信息     |
| .dynsym    动态符号表     |
| .dynstr    动态字符串表   |
| .rel/.rela 重定位表       |
+--------------------------+
```

---

# 🔶 三、重要的 ELF 段解释（so 特有的部分）

## **1️⃣ .dynamic 段（so 的核心）**

这个段告诉动态链接器：

* 需要链接哪些库（DT_NEEDED）
* 使用哪种重定位表（DT_RELA / DT_REL）
* 动态符号表在哪（DT_SYMTAB）
* 字符串表在哪（DT_STRTAB）
* 初始化函数在哪（DT_INIT_ARRAY）
* 终止函数在哪（DT_FINI_ARRAY）

几乎所有动态加载行为都依赖它。

---

## **2️⃣ .plt（Procedure Linkage Table） & .got（Global Offset Table）**

动态函数调用的关键机制。

你程序中调用 `printf()`，编译器不知道 `printf()` 在哪，所以生成跳板：

```
call printf@plt
```

流程：

1. CPU 先跳到 `.plt`
2. `.plt` 间接跳转访问 `.got`
3. 动态链接器（ld.so）会在第一次调用时修复 `.got` 的真实地址（lazy binding）

> `.plt/.got` 是 so 格式最难理解但也是最关键的机制。

---

## **3️⃣ .rel.plt / .rela.plt**

延迟绑定需要的重定位项。

---

## **4️⃣ .init_array / .fini_array**

so 加载与卸载时执行的构造函数、析构函数 C++ 依赖的东西。

---

# 🔶 四、`.so` 的编译过程（简述）

这是 so 能工作的重要基础。

```
源代码 --> .o 文件 --> 链接器 ld --> 生成 .so
```

步骤：

1. `.o` 文件是 **ET_REL（可重定位）**
2. ld 把 `.o` 和依赖库合并
3. 生成 `ET_DYN（位置无关）` 类型的 so 文件
4. 编译器一般会开启 **PIC（Position Independent Code）**
   让 so 可以被加载到任意地址（ASLR）

---

# 🔶 五、`.so` 的加载流程（重点，实战必懂）

下面分 **系统加载可执行文件** 和 **加载 so 文件** 两个路径详讲。

---

# ⭐ 六、程序启动时加载 `.so`（静态加载）

假设你运行一个程序：

```
./a.out
```

这个程序中链接了 libc.so, libm.so 之类动态库。

加载流程：

---

## **步骤 1：内核加载 ELF 主程序**

1. 内核（通过 `execve`）读取 a.out 的 **Program Header Table（PHT）**
2. 根据 `PT_LOAD` 把代码段/数据段加载到内存
3. 发现其类型是 **ET_EXEC 或 ET_DYN** 并依赖动态库（有 PT_INTERP）
4. 于是内核加载动态链接器：

```
/lib64/ld-linux-x86-64.so.2
```

---

## **步骤 2：动态链接器 ld.so 开始工作**

`ld.so` 会：

### 1. 解析主程序的 `.dynamic` 段

得到：

* 依赖哪些 so（DT_NEEDED）
* 重定位表地址
* GOT/PLT 信息

---

### 2. 递归加载所有依赖的 so

搜索路径顺序：

```
LD_LIBRARY_PATH
rpath
runpath
/etc/ld.so.cache
/lib /usr/lib 等默认目录
```

---

### 3. 将各个 `.so` 文件映射到内存（mmap）

载入时并不会先修复所有符号（lazy binding）。

---

### 4. 对所有立即绑定（non-LAZY）的符号做重定位

修复 GOT、全局变量引用等。

---

### 5. 执行 `.init_array` 中的初始化函数

如 C++ 全局对象构造。

---

### 6. 跳转到用户程序入口点（_start）

程序正式开始运行。

---

# ⭐ 七、程序运行中加载 `.so`（动态加载 dlopen）

流程更“动态”，常用于插件系统：

```
void* h = dlopen("libxxx.so", RTLD_LAZY);
```

内部流程：

1. `dlopen` 调用 ld.so 中的函数
2. ld.so 做 mmap() 把 so 映射入进程
3. 解析 `.dynamic` 段
4. 若是 `RTLD_NOW` 则立即解析所有符号
5. 若是 `RTLD_LAZY` 则用懒绑定
6. 执行 `.init_array`（比如注册插件）
7. 返回 so 句柄

查找 symbol:

```
dlsym(h, "foo");
```

---

# 🔶 八、`.so` 的懒绑定（Lazy Binding）

这是 Linux 动态库加载最关键的性能技巧。

首次调用 `func()` 时：

```
call func@plt
```

PLT 会跳到：

```
plt_stub -> ld.so -> 修复 GOT -> 跳回真正的函数
```

之后就不需要再跳转 ld.so 了，性能和直接调用一样。

---

# 🔶 九、总结（整体概念图）

下面是一个高层次的 `.so` 加载流程图，帮助你把所有概念串起来：

```
         +-------------------+
         | ELF main program  |
         +-------------------+
                   |
                   | execve
                   v
         +-------------------+
         | Kernel loads ld.so|
         +-------------------+
                   |
                   v
         +-------------------+
         | ld.so parses PHT  |
         +-------------------+
                   |
                   v
         +---------------------------+
         | Load all needed .so       |
         | (DT_NEEDED)               |
         +---------------------------+
                   |
                   v
         +---------------------------+
         | Relocation (GOT/PLT fix)  |
         +---------------------------+
                   |
                   v
         +---------------------------+
         | Run init_array constructors|
         +---------------------------+
                   |
                   v
         +---------------------------+
         |     Jump to _start        |
         +---------------------------+
```

---


