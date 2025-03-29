---
title: 编写中，未定稿
draft: true  # 设置为草稿
date: 2023-01-01 11:00:00
categories:
- 数据库
---

GopherLua是一个用Go写的lua虚拟机和编译器。GopherLua提供了API，可以让你轻松将lua脚本嵌入Go主机主机程序中。GopherLua目标是成为具有可扩展语义的脚本语言。

github地址为：https://github.com/yuin/gopher-lua#installation

## 安装

```shell
go get github.com/yuin/gopher-lua
```

## 使用示例

### 直接运行lua命令

```go
import lua "github.com/yuin/gopher-lua"

func main() {
	L := lua.NewState()
	defer L.Close()
	if err := L.DoString(`print("hello")`); err != nil {
		panic(err)
	}
}
```

运行：

```shell
go run .      
hello
```

### 运行lua文件

```go
import lua "github.com/yuin/gopher-lua"

func main() {
	L := lua.NewState()
	defer L.Close()
	if err := L.DoFile("hello.lua"); err != nil {
		panic(err)
	}
}
```

hello.lua的内容为：

```lua
print("hello")
```

## 传递参数

下面介绍如何向lua程序传递参数和接收参数

所有GopherLua里的数据都是一个LValue类型的值。LValue有以下方法可以调用：

- `String() string`
- `Type() LValueType`

LValue类型是一个interface，有很多类型实现了该interface，比如LNilType、LBool、LNumber、LString、LFunction、LUserData、LState、LTable、LChannel等等

