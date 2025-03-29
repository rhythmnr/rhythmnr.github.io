---
title: golang定义func类型
date: 2023-02-06 18:00:00
tags:
- 原创
categories:
- golang
---
如下，定义一个DecodeFunc类型，DecodeFunc是一个func([]byte, PacketBuilder)类型，可以通过d(data, p)调用

```go
type DecodeFunc func([]byte, PacketBuilder) error

// Decode implements Decoder by calling itself.
func (d DecodeFunc) Decode(data []byte, p PacketBuilder) error {
	// function, call thyself.
	return d(data, p)
}

// DecodePayload is a Decoder that returns a Payload layer containing all
// remaining bytes.
var DecodePayload Decoder = DecodeFunc(decodePayload)
func decodePayload(data []byte, p PacketBuilder) error {return nil}
```

