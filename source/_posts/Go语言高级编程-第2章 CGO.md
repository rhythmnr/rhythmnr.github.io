---
title:  Go语言高级编程-第2章 CGO
date: 2021-12-4 00:00:00
tags:
- 读书
categories:
- golang
---

> Go语言高级编程系列是我读《Go语言高级编程》时的一些要点总结。

1. **C语言作为一个 通用语言，很多库会选择提供一个C兼容的API**，然后用其他 不同的编程语言实现。**Go语言通过自带的一个叫CGO的工具来支持C语言函数调用**，同时我们可以用Go语言导出C动态库 接口给其它语言使用。本章主要讨论CGO编程中涉及的一些问 题。

2. 一个简单的CGO程序

   ```go
   package main
   import "C"
   func main() {
       println("hello cgo")
   }
   ```

   **代码通过 import "C" 语句启用CGO特性**，主函数只是通过Go 内置的println函数输出字符串，其中并没有任何和CGO相关的 代码。虽然没有调用CGO的相关函数，但是go build命令会在 编译和链接阶段启动gcc编译器，这已经是一个完整的CGO程 序了。

3. 另一个程序

   ```go
   package main
   //#include <stdio.h>
   import "C"
   func main() {
       C.puts(C.CString("Hello, World\n"))
   }
   ```

   我们不仅仅通过 import "C" 语句启用CGO特性，同时包含C 语言的 <stdio.h> 头文件。然后通过CGO包的 C.CString 函 数将Go语言字符串转为C语言字符串，最后调用CGO包的C.puts 函数向标准输出窗口打印转换后的C字符串

   相比“Hello, World 的革命”一节中的CGO程序最大的不同是: 我们没有在程序退出前释放 C.CString 创建的C语言字符串; 还有我们改用 puts 函数直接向标准输出打印，之前是采用fputs 向标准输出打印。

   没有释放使用 C.CString 创建的C语言字符串会导致内存泄 漏。但是对于这个小程序来说，这样是没有问题的，因为程序 退出后操作系统会自动回收程序的所有资源。

不写了后面的，之前的之前看不懂没看完
