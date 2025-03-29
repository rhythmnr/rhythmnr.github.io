---
title: golang将某个整数存储到文件中与读取
date: 2023-02-21 18:00:00
tags:
- 原创
categories:
- golang
---


**写入**

Go语言提供了内置的标准库，可以方便地将整数存储到文件中。您可以使用encoding/binary包进行二进制编码。

```go
mport (
	"encoding/binary"
	"log"
	"os"
)

func main() {
	// 打开文件
	file, err := os.Create("integer.bin")
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	// 写入整数
	integer := int64(123456789)
	err = binary.Write(file, binary.LittleEndian, &integer)
	if err != nil {
		log.Fatal(err)
	}
}
```

在该代码中看到，使用binary.Write方法将整数写入文件。您可以指定存储字节序，例如binary.LittleEndian或binary.BigEndian。

此外，也可以使用其他三方库，如gopkg.in/vmihailenco/msgpack.v2或github.com/tinylib/msgp，以更方便地将整数进行编码并存储到文件中。

**读取**

 ```go
import (
	"encoding/binary"
	"log"
	"os"
)

func main() {
	// 打开文件
	file, err := os.Open("integer.bin")
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	// 读取整数
	var integer int64
	err = binary.Read(file, binary.LittleEndian, &integer)
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("Read integer: %d", integer)
}
 ```

可以使用binary.Read方法从文件中读取整数，并将其存储在变量中。同样，可以指定读取字节序，以确保与写入时使用的字节序相同。

