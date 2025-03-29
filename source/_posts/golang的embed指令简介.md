---
title: golang的embed指令简介
date: 2024-2-19 10:05:00
categories:
- golang
---

embed直接翻译的意思就是嵌入，顾名思义go embed实现的效果就是嵌入指定的文件等内容到编译后的二进制可执行文件中。

## 应用场景

这个功能有很多应用场景，下面举两个例子：

### 配置文件固定

比如需要把程序的配置文件固定，又想要在运行时只运行一个二进制文件，不想再单独复制一份配置文件然后手动指定配置文件了，那么可以通过go embed将配置文件和程序打包到一个二进制文件中，运行该二进制文件即可，不需要手动指定配置文件。将程序交付给其他人时也只要交付go embed之后的二进制文件，不需要交付配置文件。不使用go embed时需要交付二进制文件和配置文件。

**注意一般嵌入的都是可读文件，不要把二进制文件嵌入，go embed并不适用二进制文件。**

## 简介

embed是golang的编译器指令 compiler directives 中的一种

golang的编译器指令是通过在代码中嵌入类似注释的命令，来控制编译器。常见的编译器指令包括：

```go
// go:linkname
// go:noinline
// go:noescape
// go:generate
```

## 示例

 下面是一个简单的例子（来源于Go By Example），这是main.go的内容：

```go
package main

import (
	"embed"
)

//go:embed folder/single_file.txt
var fileString string

//go:embed folder/single_file.txt
var fileByte []byte

//go:embed folder/*.hash
var folder embed.FS

func main() {

	print(fileString)
	print(string(fileByte))

	content1, _ := folder.ReadFile("folder/file1.hash")
	print(string(content1))

	content2, _ := folder.ReadFile("folder/file2.hash")
	print(string(content2))
}
```

文件目录为：

```shell
├── folder
│   ├── file1.hash
│   ├── file2.hash
│   └── single_file.txt
├── main.go
```

folder目录下的各个文件都有各自的内容。

运行结果为：

```shell
single_file.txt content
single_file.txt content
file1.hash content
file2.hash content
```

下面关注下代码中比较重要的部分：

```go
//go:embed folder/single_file.txt
var fileString string
```

这里通过`//go:embed folder/single_file.txt`将`folder/single_file.txt`目录嵌入了程序，并且指定了fileString来接收文件的内容。然后可以直接`print(fileString)`打印出文件的内容。

除了指定具体的文件名，还可以用通配符来匹配所有需要的文件。比如上面的：

```go
//go:embed folder/*.hash
var folder embed.FS
```

`//go:embed folder/*.hash`指定将folder目录下所有.hash文件嵌入到程序，并用embed.FS类型的folder变量接收folder目录下所有.hash文件的内容。然后可以对folder操作，读取folder中指定文件的内容：

```go
	content1, _ := folder.ReadFile("folder/file1.hash")
	print(string(content1))
```

此时可以打印出`folder/file1.hash`的内容。

除了直接go run运行，比较重要的用法就是go build编译后运行。编译后运行效果和上文相同。即使把编译后的二进制文件移动到其他位置，不移动folder目录，运行二进制文件得到的效果还是不变。仿佛通过embed指定的folder目录被嵌入到了二进制文件中，就算folder目录不是go程序。此时运行十分方便，没有额外冗余的部分了。

