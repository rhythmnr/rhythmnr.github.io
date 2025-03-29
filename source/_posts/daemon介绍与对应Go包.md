---
title: daemon介绍与对应Go包
draft: true  # 设置为草稿
date: 2023-08-15 11:00:00
categories:
- golang
---

##daemon介绍

守护进程（daemon）是一种作为后台进程连续运行并唤醒以处理定期服务请求（通常来自远程进程）的程序。 守护程序收到操作系统 (OS) 的请求警报，并且它本身响应该请求或将请求转发到适当的另一个程序或进程。

常见的守护进程包括打印后台处理程序、电子邮件处理程序和其他管理管理任务的程序。 许多 Unix 或 Linux 实用程序作为守护进程运行。 例如，在 Linux 上，网络时间协议 (NTP) 守护程序用于测量运行它的计算机上的时钟与网络上所有其他计算机的时钟之间的时间差。 每台主机上都运行一个时间守护程序，其中一台被指定为主计算机，所有其他计算机被指定为辅助计算机。 辅助守护程序首先向主时间守护程序发送请求以找出正确的网络时间，从而重置其主机上的网络时间。

守护进程是在后台运行的长期运行进程，通常不与终端交互，而是在系统启动时自动启动，并在系统关闭时自动关闭。这种方式通常用于服务器程序、监控程序等。

## go的daemon包

常用的daemon包是 https://github.com/sevlyar/go-daemon

`go-daemon` 库允许你以守护进程方式运行 Go 程序，提供以下特性：

1. **Forking 子进程：** `go-daemon` 可以 fork 出子进程，然后让子进程继续运行后续的代码，而父进程则退出。这样可以确保程序在后台持续运行。
2. **文件描述符重定向：** 在 fork 过程中，`go-daemon` 可以重定向标准输入、输出和错误流，以及其他文件描述符。这有助于将输出和日志记录到文件，同时释放终端控制。
3. **进程信号处理：** `go-daemon` 提供了对各种进程信号的处理，如重启、停止等。
4. **进程 PID 文件：** `go-daemon` 支持创建 PID 文件，用于标识守护进程的进程 ID。
5. **独立的守护进程环境：** `go-daemon` 会创建一个独立的守护进程环境，以确保程序可以在后台运行。
6. **进程退出时清理：** 守护进程退出时，`go-daemon` 可以进行一些清理工作，如删除 PID 文件等。

下面是示例代码：

```go
package main

import (
	"fmt"
	"html"
	"log"
	"net/http"

	"github.com/sevlyar/go-daemon"
)

func main() {
	cntxt := &daemon.Context{
		PidFileName: "daemon-example.pid",
		PidFilePerm: 0644,
		LogFileName: "daemon.log",
		LogFilePerm: 0640,
		WorkDir:     "./",
		Umask:       027,
		Args:        []string{"[daemon-example]"},
	}

	d, err := cntxt.Reborn()
	if err != nil {
		fmt.Println("Unable to run: ", err)
		return
	}
	if d != nil {
		return
	}
	defer cntxt.Release()

	fmt.Println("- - - - - - - - - - - - - - -")
	fmt.Println("daemon started")

	// 模拟长时间运行
	// time.Sleep(10 * time.Second)

	serveHTTP()
	fmt.Println("daemon terminated")
}

func serveHTTP() {
	http.HandleFunc("/", httpHandler)
	http.ListenAndServe("127.0.0.1:8089", nil)
}

func httpHandler(w http.ResponseWriter, r *http.Request) {
	log.Printf("request from %s: %s %q", r.RemoteAddr, r.Method, r.URL)
	fmt.Fprintf(w, "go-daemon........: %q", html.EscapeString(r.URL.Path))
}
```

运行后会在当前目录下新建两个文件：

```shell
daemon-example.pid daemon.log
```

daemon-example.pid里存储了当前的进程ID，

运行go run .后终端没有打印，但是程序其实已经在后台运行了，调用如下命令可以看到有结果输出：

```shell
curl 127.0.0.1:8089                        
go-daemon........: "/"%                                                               
```

