---
title: 关于CPU信息的一些介绍
date: 2023-8-30 00:00:00
tags:
- 原创
categories:
- 其他
---

**物理 CPU 个数**（Socket 数量）：表示物理处理器的数量，即服务器上安装了多少颗物理 CPU 芯片。

`lscpu` 命令会返回系统中的物理 CPU 的数量。

这个数量可以在返回结果中的 `Socket(s)` 参数中找到。

**CPU 核心数**：表示**每颗**物理 CPU 芯片内部的核心数量，也称为逻辑 CPU 核心数。现代的物理 CPU 芯片通常有多个核心，每个核心都可以独立执行指令。

`lscpu` 命令会返回系统中的 CPU 核心数。可以在返回结果中的 `Core(s) per socket` 参数中找到。

> 如果系统中有两颗物理 CPU，每颗 CPU 有 8 个核心，那么总共的 CPU 核心数就是 2 * 8 = 16 个。
>
> 物理 CPU 个数决定了系统中有多少颗独立的物理处理器，而每颗物理 CPU 的核心数决定了系统可以并行执行的任务数量。

**每个 CPU 核心支持的线程数**：通常是 1（无超线程）或 2（支持超线程）。通常用于表示超线程技术的支持。

`lscpu` 命令返回的Thread(s) per core: 显示每个 CPU 核心支持的线程数。

**逻辑 CPU 数**是指系统操作系统和应用程序所看到的虚拟 CPU 核心数量。与上面的`每个CPU核心支持的线程数`是有关系的。`lscpu` 命令会返回系统的逻辑 CPU 数。

在`lscpu` 命令输出中，你可以找到 `CPU(s)` 这一行，它会显示逻辑 CPU 的总数。这个值反映了系统中虚拟 CPU 核心的数量，包括通过超线程技术支持的虚拟核心。

**物理 CPU 插槽的数量**：每个 CPU 插槽可以容纳一个物理 CPU 芯片。在多 CPU 架构的系统中，可能有多个物理 CPU 插槽，每个插槽中都可以放置一个物理 CPU。每个物理 CPU 芯片上可以有多个 CPU 核心。

`lscpu` 命令中的 `Socket(s)` 表示系统中物理 CPU 插槽的数量。

> 补充：
>
> `runtime.NumCPU()` 函数返回的是当前系统可用的逻辑 CPU 核心数。这个函数返回的值并不是物理 CPU 的数量，而是操作系统可用的逻辑处理单元（线程）的数量。在多核心的处理器上，每个核心都可以有多个线程，因此 `runtime.NumCPU()` 返回的是逻辑 CPU 核心数。

下面是一个世纪运行过程中lscpu的输出，可以作为参考：

```shell
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                16
On-line CPU(s) list:   0-15
Thread(s) per core:    1
Core(s) per socket:    1
Socket(s):             16
NUMA node(s):          1
Vendor ID:             AuthenticAMD
CPU family:            23
Model:                 1
Model name:            AMD EPYC Processor (with IBPB)
Stepping:              2
CPU MHz:               2894.560
BogoMIPS:              5789.12
Hypervisor vendor:     KVM
Virtualization type:   full
L1d cache:             32K
L1i cache:             64K
L2 cache:              512K
L3 cache:              8192K
NUMA node0 CPU(s):     0-15
```

