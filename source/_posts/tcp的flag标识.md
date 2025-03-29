---
title: tcp的flag标识
date: 2023-2-13 20:33:00
categories:
- 计算机网络
---
## 所有标识位

图片来自[维基百科](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)

[![pSWuNHs.png](https://s1.ax1x.com/2023/02/09/pSWuNHs.png)](https://imgse.com/i/pSWuNHs)

TCP报文段分为首部和数据部分，TCP首部分为固定部分和选项(变长部分)，首部的固定部分(共有20字节);可选部分的长度为0~40字节

### 6个主要的标志位

> URG和PSH在数据传输的过程中使用

紧急位URG

确认位ACKACK 携带两个信息。期待要收到下一个数据包的编号、接收方的接收窗口的剩余容量

推送位PSH(Push)。PSH 标志位表示发送端请求接收端尽快将数据传递给应用层。

复位位RST (Reset)：RST= 1时，表明TCP连接中出现严重差错（如主机崩溃或其他原因)，必须释放连接，然后再重新建立运输连接。

同步位SYN：表示这是一个连接请求或连接接受报文。

终止位FIN：表明此报文段的发送方的数据已发送完毕，并要求释放传输连接。

###**另外3个**

CWR：告诉对方本端已经减小了拥赛窗口

ECE：告诉对端对端的网络发生的拥赛

NS：无报文段

## 常见的三次握手四次挥手

### 三次握手

[![pSRjFr4.png](https://s1.ax1x.com/2023/02/09/pSRjFr4.png)](https://imgse.com/i/pSRjFr4)

SYN=1

SYN=1 ACK=1

ACK=1

### 四次挥手

[![pSRjXQO.png](https://s1.ax1x.com/2023/02/09/pSRjXQO.png)](https://imgse.com/i/pSRjXQO)

FIN=1

ACK= 1

FIN=1，ACK= 1

ACK = 1

这是wireshark里的抓到的三次握手四次挥手，可以观察具体场景中的Flag设置情况

[![pSoehKf.png](https://s1.ax1x.com/2023/02/13/pSoehKf.png)](https://imgse.com/i/pSoehKf)

我抓到的Flags有9位：

[![pSRv3lT.png](https://s1.ax1x.com/2023/02/09/pSRv3lT.png)](https://imgse.com/i/pSRv3lT)

## 所有常见的flag组合

根据我电脑上抓包的结果，共18项

```
20:ACK+RST 重置连接，并且确认了已经收到了对端的消息
18:ACK+SYN 确认连接建立
25:ACK+PSH+FIN 要求对端尽快将数据传递给应用层且关闭连接
153:CWR+ACK+PSH+FIN 正在结束连接，并且请求对端尽快将数据传递给应用层，通知对端本端已经减小了拥戴窗口
144:CWR+ACK 告诉对端本端已经减小了拥戴窗口
194:CWR+ECE+SYN 请求建立TCP连接，告诉对端本端已经减小了拥戴窗口且对端网络拥塞了
152:CWR+ACK+PSH 收到了发送端传输的数据，并请求对端紧急发送数据，告诉对端本端已经减小了拥戴窗口
24:ACK+PSH 确认了已经收到了对端尽快将数据传递给应用层，如发送HTTP请求和HTTP响应中会出现
16:ACK 确认数据已成功接收
17:ACK+FIN 关闭连接
2:SYN 建立连接
82:ECE+ACK+SYN TCP连接正在建立，通知对端其网络拥塞了
4:RST 重置连接
148:CWR+ACK+RST 强制重置连接且通知发送方网络拥塞的情况。通常在网络出现故障或拥塞时使用，用于快速终止连接。
34:URG+SYN 在连接建立时发送紧急数据
210:CWR+ECE+ACK+SYN TCP的ECN (显式拥塞通知)功能被启用，同时也表示TCP连接正在建立。
146:CWR+ACK+SYN 发送方已经接收到对方的SYN请求，并且表示接收到对方的拥塞窗口减少通知，同时也向对方发送一个确认应答。
26:ACK+PSH+SYN 发送方已经接收到对方的SYN请求，同时需要立即推送数据，并向对方发送一个确认应答。
```

没有URG的包，根据chatgpt的回答，增加了以下这些URG的情况：

> 来自chatgpt
>
> 根据TCP协议的标准（RFC 793），所有合法的TCP标志组合如下：
>
> 1. ACK：一般用于确认数据已成功接收。
> 2. SYN：用于建立连接。
> 3. SYN-ACK：连接请求后，服务端会返回SYN-ACK标志以确认连接建立。
> 4. **FIN：用于关闭连接。**（这个上面的没有）
> 5. FIN-ACK：在关闭连接请求后，服务端会返回FIN-ACK标志以确认连接关闭。
> 6. RST：用于重置连接。
> 7. ACK-RST：表示重置请求已确认。
> 8. **PSH：用于将数据强制刷到接收端，以避免缓存。**
> 9. **URG：用于标识数据有紧急标志。**
> 10. **ECE：拥塞状态下的TCP的标志。**
> 11. **CWR：拥塞状态下的TCP的标志。**
> 12. NS：用于识别非恶意的报文。(我没有统计这个，忽略这个)

```
56:URG+PSH+ACK 强制将紧急数据立即传递给应用层
48:URG+ACK 优先处理紧急数据
```

总结下来就是如下这些TCP标志位（应该是大部分的，可能不是全部的）和对应的含义：

```
20: ACK、RST被设置，表示重置连接，并且确认了已经收到了对端的消息
18: ACK、SYN被设置，表示确认连接建立
25: ACK、PSH、FIN被设置，表示要求对端尽快将数据传递给应用层且关闭连接
153: CWR、ACK、PSH、FIN被设置，表示正在结束连接，并且请求对端尽快将数据传递给应用层，通知对端本端已经减小了拥戴窗口
144: CWR、ACK被设置，表示告诉对端本端已经减小了拥戴窗口
194: CWR、ECE、SYN被设置，表示请求建立TCP连接，告诉对端本端已经减小了拥戴窗口且对端网络拥塞了
152 :CWR、ACK、PSH被设置，表示收到了发送端传输的数据，并请求对端紧急发送数据，告诉对端本端已经减小了拥戴窗口
24: ACK、PSH被设置，表示确认了已经收到了对端尽快将数据传递给应用层，如发送HTTP请求和HTTP响应中会出现
16: ACK被设置，表示确认数据已成功接收
17: ACK、FIN被设置，表示关闭连接
2: SYN被设置，表示建立连接
82: ECE、ACK、SYN被设置，表示TCP连接正在建立，通知对端其网络拥塞了
4: RST被设置，表示重置连接
148:CWR+ACK+RST 强制重置连接且通知发送方网络拥塞的情况。通常在网络出现故障或拥塞时使用，用于快速终止连接。
34:URG+SYN 在连接建立时发送紧急数据
210:CWR+ECE+ACK+SYN TCP的ECN (显式拥塞通知)功能被启用，同时也表示TCP连接正在建立。
146:CWR+ACK+SYN 发送方已经接收到对方的SYN请求，并且表示接收到对方的拥塞窗口减少通知，同时也向对方发送一个确认应答。
26:ACK+PSH+SYN 发送方已经接收到对方的SYN请求，同时需要立即推送数据，并向对方发送一个确认应答。
```

## 不常见的标识

### CWR和ECE

这2个在TCP拥塞控制中使用。

TCP拥塞控制通过计算窗口大小，动态调整发送速率来实现。在网络拥塞时，窗口大小会减小，减缓数据的发送速率；在网络空闲时，窗口大小会增加，加快数据的发送速率。通过这种方式，TCP拥塞控制能够在网络拥塞状态下有效地避免数据的丢失。

[参考](https://www.catchpoint.com/blog/ece-cwr-tcp)

这两个 TCP 标志与 IP 标头中的两个标志（ECT 和 CE）一起使用，以警告发送方网络拥塞，从而避免数据包丢失和重传。

Prior to acknowledging the receipt of the packet, the receiver sets the ECE flag in the TCP header of the ACK and sends it back to the sender. The sender having received the ECE marked ACK responds by halving the send window and reducing the slow start threshold.

[参考](https://www.geeksforgeeks.org/working-of-explicit-congestion-notification/)（这个讲得很好）这个讲了TCP拥塞控制的整个过程是什么样子的。

0. ECN协商
1. 现在发件人正在向另一方发送数据包。 它将在 IP 标头中设置 ECT 以通知路由器发送方和接收方都启用了 ECN，并且他们已经相互通话。
2. 路由器检测到了当前网络遇到了拥塞，当数据包到达路由器时，检测到数据包的IP头部有ECT，路由设置EC。CE 表明路由器正在经历拥塞。 CE 从路由器发送到接收方。这个过程中标头的变化是ECT [0 1] -> CE [1 1]
3. 接收方看到 IP 报头中的 CE 标记，就知道路由器拥塞了。 现在接收方有责任通知发送方拥塞。接收方标记TCP的ECE标头。

   当 SYN/ACK 位打开时，ECE 位表示 ECN 协商（也就是步骤0），而不是拥塞。 但是现在，ACK 数据包中的 ECE 位处于打开状态，因此发送方知道路由器拥塞。
4. 发送方收到了ECE标志后，发送方将其拥塞窗口减少一半。
5. 发送方在TCP头中设置CWR，在IP头中设置ECT，告诉接收方自己已经减小了拥塞窗口为了避免拥塞。
6. 当数据包到达路由器时，如果路由器感到拥塞，它可以将 ECT 推销给 CE。
7. 当数据包到达接收方时，因为这个数据包有CWR标识，接收方检测到这个标识后会停止在后续的 ACK 数据包中向发送方发送 ECE 标志。

### PSH和URG

[参考](https://packetlife.net/blog/2011/mar/2/tcp-flags-psh-and-urg/)

初始 HTTP 请求设置了 PSH 标志，表明发送方没有进一步的数据要添加，接收方应立即发送到应用程序，不要把数据放到TCP缓冲区了。

URG用来告诉接收方有些数据是紧急的并且应该被优先处理。如果设置了 URG 标志，将评估紧急指针，这是 TCP 标头中的一个 16 位字段。 该指针指示数据包中有多少数据（从第一个字节开始计算）是紧急的。

[![pSoVGsU.png](https://s1.ax1x.com/2023/02/13/pSoVGsU.png)](https://imgse.com/i/pSoVGsU)

这个URG的情况很少出现。

## RFC文档

可以查看[rfc793文档](https://datatracker.ietf.org/doc/html/rfc793)里的各个Figure，获取关于TCP连接状态变更和Flag变更的图。

## 打印某个flag对应所有被设置的位

```go
func main() {
	type FlagName struct {
		Flag int
		Name string
	}
	const (
		FlagFIN = 1 << iota
		FlagSYN
		FlagRST
		FlagPSH
		FlagACK
		FlagURG
		FlagECE
		FlagCWR
	)
	flagList := []FlagName{
		{FlagFIN, "FIN"},
		{FlagSYN, "SYN"},
		{FlagRST, "RST"},
		{FlagPSH, "PSH"},
		{FlagACK, "ACK"},
		{FlagURG, "URG"},
		{FlagECE, "ECE"},
		{FlagCWR, "CWR"},
	}
	all := []int{0x014, 0x012, 0x019, 0x099, 0x090, 0x0c2, 0x098, 0x018, 0x010, 0x011, 0x002, 0x052, 0x004}
	// all := []int{20, 18, 25, 153, 144, 194, 152, 24, 16, 17, 2, 82, 4}
	for _, i := range all {
		fmt.Printf("%d:", i)
		for index := range flagList {
			last := len(flagList) - 1 - index
			current := flagList[last]
			if current.Flag&i != 0 {
				fmt.Print(current.Name + " ")
			}
		}
		fmt.Println()
	}
}
func IntToByteArray(num int) []byte {
	size := int(unsafe.Sizeof(num))
	arr := make([]byte, size)
	for i := 0; i < size; i++ {
		byt := *(*uint8)(unsafe.Pointer(uintptr(unsafe.Pointer(&num)) + uintptr(i)))
		arr[i] = byt
	}
	return arr
}
```

比如打印wireshark抓到的flags对应的0x010，就可以查看有哪些位被设置了。

[![pSoqqtH.png](https://s1.ax1x.com/2023/02/14/pSoqqtH.png)](https://imgse.com/i/pSoqqtH)

