---
title: go语言学习-iota和左移右移的用法
date: 2022-2-15 00:00:00
tags:
- 原创
categories:
- golang
---

[原文](https://studygolang.com/articles/18059)

在go语言中iota比较特殊，是一个被编译器修改的常量，在每一个const关键字出现时被重置为0，然后在下一个const出现之前，每出现一次iota，其所代表的数字就会自动加1

左移运算符"<<“是双目运算符。**左移n位就是乘以2的n次方**。 其功能把”<<“左边的运算数的各二进位全部左移若干位，由”<<"右边的数指定移动的位数，高位丢弃，低位补0。

右移运算符">>“是双目运算符。右移n位就是除以2的n次方。 其功能是把”>>“左边的运算数的各二进位全部右移若干位，”>>"右边的数指定移动的位数。

注意：

```go
 const(
  c1 = iota  //c1=0
  c2 = iota  //c2=1
  c3 = iota  //c3=2
 )
```

```go
package iota

import "fmt"

func Test()  {

 const(
  c1 = iota  //c1=0
  c2 = iota  //c2=1
  c3 = iota  //c3=2
 )

 fmt.Println("c1 = ",c1," c2 = ",c2," c3 = ",c3,"\n")

 const(
  a = 1 << iota //a = 1
  b = 1 << iota //b = 2
  c = 1 << iota //c = 4
 )
 fmt.Println("a = ",a," b = ",b," c = ",c,"\n")

 const(
  v1 = iota //v1 = 0
  v2        //v2 = 1
  v3        //v3 = 2
 )
 fmt.Println("v1 = ",v1," v2 = ",v2," v3 = ",v3,"\n")

 const(
  x = 1 <<iota //x = 1
  y     //y = 2
  z             //z = 4
 )
 fmt.Println("x = ",x," y = ",y," z = ",z)
}
```
