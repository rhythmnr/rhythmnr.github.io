---
title: libpcap抓包时遇到权限问题
date: 2023-08-23 01:00:00
categories:
- 网络
---

我写了个非常简单的抓包程序，如下：

```go
package main

import (
	"fmt"
	"time"

	"github.com/google/gopacket"
	"github.com/google/gopacket/pcap"
)

func main() {
	handle, err := pcap.OpenLive("en0", 1024, true, -1*time.Second)
	if err != nil {
		fmt.Println(".............", err)
		return
	}
	defer handle.Close()

	packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
	packetSource.DecodeOptions = gopacket.DecodeOptions{Lazy: true, NoCopy: true,
		DecodeStreamsAsDatagrams: true}
	for s := range packetSource.Packets() {
		fmt.Println("....Timestamp.....", s.Metadata().Timestamp)
		fmt.Println(".....CaptureInfo....", s.Metadata().CaptureInfo)
	}
}
```

但是执行的时候报错了：

```shell
CGO_ENABLED=1  go run .
# github.com/google/gopacket/aaa
ld: warning: -no_pie is deprecated when targeting new OS versions
ld: warning: non-standard -pagezero_size is deprecated when targeting macOS 13.0 or later
............. en0: You don't have permission to capture on that device ((cannot open BPF device) /dev/bpf0: Permission denied)
```

显然是权限不够的问题，下面执行命令加上权限：

```shell
sudo chmod o+r /dev/bpf*
```

这个命令就是给/dev/bpf*文件加上其他用户的读权限，o表示其他用户，r标识读权限。

执行完毕后再次执行`CGO_ENABLED=1  go run .`就可以执行成功了。之前在wireshark没有加权限时wireshark也不能抓包提示权限不够，但是执行后wireshark就可以正常抓到包了。

因为我是在macOS上运行的，在 macOS 上，BPF 设备文件位于 `/dev/bpf*`，这些文件默认情况下只允许 root 用户和某些特权组的用户访问。


