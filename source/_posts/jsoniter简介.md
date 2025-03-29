---
title: jsoniter简介
date: 2022-2-7 00:00:00
tags:
- 原创
categories:
- 其他
---

## 为什么要研究这玩意？

发现的一些代码是这样写的：

```go
 // 通知连接成功
 var successReq struct {
  Type  string `json:"type"`
  Event string `json:"event"`
  User  struct {
   UserID int64 `json:"user_id"`
  } `json:"user"`
 }
 successReq.Type = typeUser
 successReq.Event = eventConnect
 successReq.User.UserID = wsConn.User.ID()

 reqContext, err := json2.Marshal(successReq)
 if err != nil {
  panic(err)
 }
```

```go
package im

import (
 "encoding/json"
 "fmt"
 "net"
 "sync/atomic"
 "time"

 "github.com/google/uuid"
 "github.com/gorilla/websocket"
 jsoniter "github.com/json-iterator/go"
 "github.com/json-iterator/go/extra"
 "go.mongodb.org/mongo-driver/bson"
)

func init() {
 extra.RegisterFuzzyDecoders()
}

var json2 = jsoniter.ConfigCompatibleWithStandardLibrary
```

可以看到`github.com/json-iterator`里的包在这里的两个用处：

```go
extra.RegisterFuzzyDecoders()
var json2 = jsoniter.ConfigCompatibleWithStandardLibrary
```

## jsoniter简介

### 性能

一个高性能 100% 兼容的“encoding/json”替代品

性能测试结果如下

![Xnip2022-03-19_15-40-48.jpg](http://tva1.sinaimg.cn/large/006gLprLgy1h0f8pue5vzj30uu0h6gmh.jpg)

### 使用方法

Replace

```
import "encoding/json"
json.Marshal(&data)
```

with

```
import "github.com/json-iterator/go"

var json = jsoniter.ConfigCompatibleWithStandardLibrary
json.Marshal(&data)
```

Replace

```
import "encoding/json"
json.Unmarshal(input, &data)
```

with

```
import "github.com/json-iterator/go"

var json = jsoniter.ConfigCompatibleWithStandardLibrary
json.Unmarshal(input, &data)
```

### 文档

包 jsoniter 实现了 RFC 4627 中定义的 JSON 的编码和解码，并提供了与标准 lib encoding/json 语法相同的接口。 从 encoding/json 转换为 jsoniter 只不过是将包替换为 jsoniter 和变量类型声明（如果有的话）。 jsoniter 接口与使用标准库的代码提供 100% 的兼容性。

“JSON and Go”（<https://golang.org/doc/articles/json_and_go.html>）描述了 Marshal/Unmarshal 如何在任意或预定义的 json 对象和字节之间进行操作，它适用于 jsoniter.Marshal/Unmarshal。

此外， jsoniter.Iterator 提供了一组不同的接口迭代给定的字节/字符串/读取器并一个一个地产生解析的元素。 这组接口根据需要读取输入并提供更好的性能。

此外：RegisterFuzzyDecoders的作用

```go
// RegisterFuzzyDecoders decode input from PHP with tolerance.
// It will handle string/number auto conversation, and treat empty [] as empty struct.
func RegisterFuzzyDecoders() 
```

详见单元测试：

```go
func Test_any_to_string(t *testing.T) {
 should := require.New(t)
 var val string
 should.Nil(jsoniter.UnmarshalFromString(`"100"`, &val))
 should.Equal("100", val)
 should.Nil(jsoniter.UnmarshalFromString("10", &val))
 should.Equal("10", val)
 should.Nil(jsoniter.UnmarshalFromString("10.1", &val))
 should.Equal("10.1", val)
 should.Nil(jsoniter.UnmarshalFromString(`"10.1"`, &val))
 should.Equal("10.1", val)
 should.NotNil(jsoniter.UnmarshalFromString("{}", &val))
 should.NotNil(jsoniter.UnmarshalFromString("[]", &val))
}
func Test_any_to_int64(t *testing.T) {
 should := require.New(t)
 var val int64

 should.Nil(jsoniter.UnmarshalFromString(`"100"`, &val))
 should.Equal(int64(100), val)
 should.Nil(jsoniter.UnmarshalFromString(`"10.1"`, &val))
 should.Equal(int64(10), val)
 should.Nil(jsoniter.UnmarshalFromString(`10.1`, &val))
 should.Equal(int64(10), val)
 should.Nil(jsoniter.UnmarshalFromString(`10`, &val))
 should.Equal(int64(10), val)
 should.Nil(jsoniter.UnmarshalFromString(`""`, &val))
 should.Equal(int64(0), val)

 // bool part
 should.Nil(jsoniter.UnmarshalFromString(`false`, &val))
 should.Equal(int64(0), val)
 should.Nil(jsoniter.UnmarshalFromString(`true`, &val))
 should.Equal(int64(1), val)

 should.Nil(jsoniter.UnmarshalFromString(`-10`, &val))
 should.Equal(int64(-10), val)
 should.NotNil(jsoniter.UnmarshalFromString("{}", &val))
 should.NotNil(jsoniter.UnmarshalFromString("[]", &val))
 // large float to int
 should.NotNil(jsoniter.UnmarshalFromString(`1234512345123451234512345.0`, &val))
}
```

重点是下面这个：

```go
func TestRRR(t *testing.T) {
 should := require.New(t)
 var val struct{}
 should.Nil(jsoniter.UnmarshalFromString(`[]`, &val))
 should.Equal(struct{}{}, val)
}
```

如果没有调用RegisterFuzzyDecoders会报错：

```shell
--- FAIL: TestRRR (0.00s)
    /Users/rhettnina/mylocalfile/同步（坚果云）/code/work/live-service/controllers/im/conn_test.go:582: 
         Error Trace: conn_test.go:582
         Error:       Expected nil, but got: &errors.errorString{s:"skipObjectDecoder: expect object or null, error found in #0 byte of ...|[]|..., bigger context ...|[]|..."}
         Test:        TestRRR
FAIL
```

## 例子

```go
package main

import (
 "encoding/json"
 "fmt"

 jsoniter "github.com/json-iterator/go"
 "github.com/json-iterator/go/extra"
)

var json2 = jsoniter.ConfigCompatibleWithStandardLibrary

func init() {
 extra.RegisterFuzzyDecoders()
}

type input struct {
 Code string `json:"code"`
 Xa   string `json:"xa"`
}

func main() {
 i := input{
  Code: "123",
  Xa:   "456",
 }
 v, err := json.Marshal(i)
 fmt.Println("-------string(v)---", string(v))
 var result struct {
  Code int   `json:"code"`
  Xa   int64 `json:"xa"`
 }
 err = json2.Unmarshal([]byte(string(v)), &result)
 fmt.Println("----Unmarshal----", err)
 fmt.Println("-----result--", result)
}
```

运行结果如下：

```shell
[Running] go run "/Users/rhettnina/mylocalfile/同步（坚果云）/code/work/live-service/tt/main.go"
-------string(v)--- {"code":"123","xa":"456"}
----Unmarshal---- <nil>
-----result-- {123 456}

[Done] exited with code=0 in 1.231 seconds
```

去掉`extra.RegisterFuzzyDecoders()`会报错，可见RegisterFuzzyDecoders会兼容string转int的情况：

```shell
[Running] go run "/Users/rhettnina/mylocalfile/同步（坚果云）/code/work/live-service/tt/main.go"
-------string(v)--- {"code":"123","xa":"456"}
----Unmarshal---- struct { Code int "json:\"code\""; Xa int64 "json:\"xa\"" }.Code: readUint64: unexpected character: �, error found in #9 byte of ...|{"code":"123","xa":|..., bigger context ...|{"code":"123","xa":"456"}|...
-----result-- {0 0}
```

加了个In字段测试int转string，发现还是可以正常运行，RegisterFuzzyDecoders还是挺有用的：

```go
package main

import (
 "encoding/json"
 "fmt"

 jsoniter "github.com/json-iterator/go"
 "github.com/json-iterator/go/extra"
)

var json2 = jsoniter.ConfigCompatibleWithStandardLibrary

func init() {
 extra.RegisterFuzzyDecoders()
}

type input struct {
 Code string `json:"code"`
 Xa   string `json:"xa"`
 In   int64  `json:"in"`
}

func main() {
 i := input{
  Code: "123",
  Xa:   "456",
  In:   100,
 }
 v, err := json.Marshal(i)
 fmt.Println("-------string(v)---", string(v))
 var result struct {
  Code int    `json:"code"`
  Xa   int64  `json:"xa"`
  In   string `json:"in"`
 }
 err = json2.Unmarshal([]byte(string(v)), &result)
 fmt.Println("----Unmarshal----", err)
 fmt.Println("-----result--", result)
}
```

```shell
-------string(v)--- {"code":"123","xa":"456","in":100}
----Unmarshal---- struct { Code int "json:\"code\""; Xa int64 "json:\"xa\""; In string "json:\"in\"" }.Code: readUint64: unexpected character: �, error found in #9 byte of ...|{"code":"123","xa":|..., bigger context ...|{"code":"123","xa":"456","in":100}|...
-----result-- {0 0 }
```
