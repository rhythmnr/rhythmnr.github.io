---
title: golang实现二进制位
date: 2023-02-21 18:00:00
tags:
- 原创
categories:
- golang
---

```go
type Bit int16

// 设置第i位置为1
func (b *BitMask) Set(i int) {
	*b |= (1 << (i - 1))
}

// 判断是否第i位是否为1
func (b BitMask) IsSet(i int) bool {
	return b&(1<<(i-1)) != 0
}

// 设置第i位置为0，即取消设置第i位
func (b *BitMask) Unset(i int) {
	*b &^= (1 << (i - 1))
}

```
