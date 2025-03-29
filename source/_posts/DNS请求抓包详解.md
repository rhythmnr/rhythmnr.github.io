---
title: DNS抓包请求详解
date: 2023-2-24 17:14:00
categories:
- 计算机网络
---

> DNS的传输层使用的是**UDP协议或者TCP协议。**
>
> DNS在某些情况下会使用TCP协议。DNS通常使用UDP协议（用户数据报协议）来进行域名查询，因为UDP速度快、效率高，而且查询数据包通常较小，适合在UDP上进行。但是，当DNS查询数据包的大小超过了UDP数据包的最大限制时（通常是512字节），DNS会使用TCP协议来进行查询。此外，某些DNS服务器也会将DNS查询请求强制转换为TCP协议，以增加安全性和可靠性。

## Transaction ID

在DNS协议中，每个DNS请求报文都包含一个16位的Transaction ID字段，用于标识该DNS请求和对应的DNS响应。**Transaction ID由客户端随机生成，并且在DNS响应报文中原样返回，以便客户端能够将响应与请求进行匹配**。Transaction ID的生成没有特定的规律，通常是使用伪随机数生成算法来生成。

## DNS解析过程

### 终端输出

执行`dig +trace www.baidu.com`返回的结果为：

<details><summary>展开/收起</summary>
dig +trace www.baidu.com<br>
<br>
; <<>> DiG 9.10.6 <<>> +trace www.baidu.com<br>
;; global options: +cmd<br>
.			255946	IN	NS	f.root-servers.net.<br>
.			255946	IN	NS	g.root-servers.net.<br>
.			255946	IN	NS	h.root-servers.net.<br>
.			255946	IN	NS	b.root-servers.net.<br>
.			255946	IN	NS	j.root-servers.net.<br>
.			255946	IN	NS	l.root-servers.net.<br>
.			255946	IN	NS	k.root-servers.net.<br>
.			255946	IN	NS	e.root-servers.net.<br>
.			255946	IN	NS	a.root-servers.net.<br>
.			255946	IN	NS	c.root-servers.net.<br>
.			255946	IN	NS	m.root-servers.net.<br>
.			255946	IN	NS	i.root-servers.net.<br>
.			255946	IN	NS	d.root-servers.net.<br>
.			255946	IN	RRSIG	NS 8 0 518400 20230305170000 20230220160000 951 . xU5kKOHDoGfyQVeW8Huv74NJxyAaKyuDUxli5P0K08z+vvLlA3N87OVL bhwoihWpKg+Of2S2nI6jx80n+S2Ty74PSOijnu9sW7vvTD30Wg1Vgtda rrOc2gf9UmDkT+mLda/IiX7DtAla76k9t+owykaxyPHfdkLH1cfZmGj6 0KPRJDf9gnopBomQIBa4/m3UqwJlefYA0Wr01Bs+BaCKHPQEnrBSsBYJ jBbFOkadtuXtVym5Bapg8TYJQQfJxlutIuuEyC1BDSqw33poKUcKzXg3 v7VIeHSq+wKaOyDt7m9wrCxOhj3uZurNO+b88xmtV4pX+HxRGs+Yx2QY kCGskQ==<br>
;; Received 1097 bytes from 10.50.4.107#53(10.50.4.107) in 12 ms<br>
<br>
com.			172800	IN	NS	m.gtld-servers.net.<br>
com.			172800	IN	NS	k.gtld-servers.net.<br>
com.			172800	IN	NS	c.gtld-servers.net.<br>
com.			172800	IN	NS	i.gtld-servers.net.<br>
com.			172800	IN	NS	h.gtld-servers.net.<br>
com.			172800	IN	NS	b.gtld-servers.net.<br>
com.			172800	IN	NS	e.gtld-servers.net.<br>
com.			172800	IN	NS	f.gtld-servers.net.<br>
com.			172800	IN	NS	d.gtld-servers.net.<br>
com.			172800	IN	NS	g.gtld-servers.net.<br>
com.			172800	IN	NS	j.gtld-servers.net.<br>
com.			172800	IN	NS	a.gtld-servers.net.<br>
com.			172800	IN	NS	l.gtld-servers.net.<br>
com.			86400	IN	DS	30909 8 2 E2D3C916F6DEEAC73294E8268FB5885044A833FC5459588F4A9184CF C41A5766<br>
com.			86400	IN	RRSIG	DS 8 1 86400 20230308170000 20230223160000 951 . eiW76HnA6iWoL7HVQQlR5GwBc3QagvdHZJEH3d13ze+ukMwieArh0Ec2 pd10S4nFcW8S37ajG+u2tzLV09ZlXhpRNbXX84qD17QvWeh7IeT+vCFl lk4I/+55UcVdI3LfhHmJPF4cjMLQsIbKCB1Ryf+sWQsVKYE9XeoUR5Mp rV7W8I1UWn9e1yMs5C+ujrd/UFPb/Uw4QS6RZ7Q0K39XOUqFRqUuSnFQ Klg5fExB7I6/dARBqK4lnOzMssrF97HmGtGNUkpb3CgBMIwSDGriM+Uf vQVRzFyhGNHD3KigUV18uYw8Yqq48By+BKJmzMd4IQGiy0e0WNavs2vd 9Ph/+A==<br>
;; Received 1176 bytes from 192.5.5.241#53(f.root-servers.net) in 27 ms<br>
<br>
baidu.com.		172800	IN	NS	ns2.baidu.com.<br>
baidu.com.		172800	IN	NS	ns3.baidu.com.<br>
baidu.com.		172800	IN	NS	ns4.baidu.com.<br>
baidu.com.		172800	IN	NS	ns1.baidu.com.<br>
baidu.com.		172800	IN	NS	ns7.baidu.com.<br>
CK0POJMG874LJREF7EFN8430QVIT8BSM.com. 86400 IN NSEC3 1 1 0 - CK0Q2D6NI4I7EQH8NA30NS61O48UL8G5  NS SOA RRSIG DNSKEY NSEC3PARAM<br>
CK0POJMG874LJREF7EFN8430QVIT8BSM.com. 86400 IN RRSIG NSEC3 8 2 86400 20230227052255 20230220041255 36739 com. YJw2xFEhDVUfIlym8yUrXw8rVYLxS+e/EkIJVmOkBANnfCmNPVATcGuM /DIrUz8PTWTezM5z6f2tM+KnzzXYMNL1ScDIgO/jaJUrs4aOz1EOPwD4 hk5rJ/pRSY9C87vRoxqdryDIHxg3TwwEfQglqQ9hk+P1qvU7qY5nd0yc tO+IV8Vqd0sRiteg/P1h6Bpp79v/kZNjntRTdnWLI2oW2g==<br>
HPVV07LPQ3T8RQS9HETLBJF268LK3OQ2.com. 86400 IN NSEC3 1 1 0 - HPVV8SARM2LDLRBTVC5EP1CUB1EF7LOP  NS DS RRSIG<br>
HPVV07LPQ3T8RQS9HETLBJF268LK3OQ2.com. 86400 IN RRSIG NSEC3 8 2 86400 20230301070223 20230222055223 36739 com. Z4AFqvwBHDHfzuf37WTbdeC0SMt15I4qG2MUUhjv14m7MJ6AwDCHVXAZ tqi9T+xttD0xo0wgL5pRciXRTdNPUFBq/mu5Fimp/tfosWjcshv50+4c TNdbMXubTw4yr7DLDlQGO5a2cSa6T/HrDnpdDjZJcfVYLGZP1k5nFXhc +TKjOrScSPq9lG3hH7n7SGxCTrp7j+tCjzrHjH8egq2tKg==<br>
;; Received 849 bytes from 192.5.6.30#53(a.gtld-servers.net) in 187 ms<br>
<br>
www.baidu.com.		1200	IN	CNAME	www.a.shifen.com.<br>
;; Received 72 bytes from 110.242.68.134#53(ns1.baidu.com) in 29 ms<br>

</details>

### 抓包

下面是对应的执行抓到的包：

首先访问192.5.5.241，192.5.5.241返回了**13台通用顶级域名.com服务器**的信息，如下：

[![pSz3zXn.png](https://s1.ax1x.com/2023/02/24/pSz3zXn.png)](https://imgse.com/i/pSz3zXn)

ID 81中Additional records比Authoritative nameservers多了返回addr，他们的IP地址包括IPv4和IPv6，这俩可以通过type区分，type分别为A和AAAA。ID81对应的请求为ID80：`Standard query. 0x2935 A www.baidu.com OPT`

然后选择了其中的一台，这里选择了a.gtld-servers.net，地址为192.5.6.30，192.5.6.30返回的内容如下：

[![pSzGdM9.png](https://s1.ax1x.com/2023/02/24/pSzGdM9.png)](https://imgse.com/i/pSzGdM9)

然后选择了其中的一台，这里选择了ns1.baidu.com，地址为110.242.68.134，110.242.68.134返回的内容如下：

[![pSzGxZq.png](https://s1.ax1x.com/2023/02/24/pSzGxZq.png)](https://imgse.com/i/pSzGxZq)

## DNS响应时间

### 响应和请求相邻时

首先调整下wireshark的显示设置：

[![pSzaS8f.png](https://s1.ax1x.com/2023/02/24/pSzaS8f.png)](https://imgse.com/i/pSzaS8f)

因为发现本次的dns查询的请求和响应都是相邻的，所以响应包里这个设置显示的时间就是响应时间。

wireshark抓到的包中可以看到响应时间（发出包和接收到这个包的时间差），wireshark抓到的包结果是这样的：

![pSza6it.png](https://s1.ax1x.com/2023/02/24/pSza6it.png)

命令行返回的结果为：

```shell
dig +trace csdn.com +time=1 +noall +stats

; <<>> DiG 9.10.6 <<>> +trace csdn.com +time=1 +noall +stats
;; global options: +cmd
;; Query time: 51 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Feb 24 15:11:09 CST 2023
;; MSG SIZE  rcvd: 525

;; Query time: 193 msec
;; SERVER: 198.41.0.4#53(198.41.0.4)
;; WHEN: Fri Feb 24 15:11:09 CST 2023
;; MSG SIZE  rcvd: 1168

;; Query time: 195 msec
;; SERVER: 192.54.112.30#53(192.54.112.30)
;; WHEN: Fri Feb 24 15:11:09 CST 2023
;; MSG SIZE  rcvd: 946

;; Query time: 12 msec
;; SERVER: 47.118.199.202#53(47.118.199.202)
;; WHEN: Fri Feb 24 15:11:09 CST 2023
;; MSG SIZE  rcvd: 53
```

可以看到4个DNS请求的响应时间在wireshark都有对应的结果，包括SERVER字段和wireshark里的也是一致的。

### 响应和请求分隔时

试着执行了下命令：

```shell
dig +trace csdn.com +time=1 +noall +stats &
dig +trace zhihu.com +time=1 +noall +stats
```

运行结果为：

<details><summary>展开/收起</summary>
; <<>> DiG 9.10.6 <<>> +trace zhihu.com +time=1 +noall +stats<br>
;; global options: +cmd<br>
<br>
; <<>> DiG 9.10.6 <<>> +trace csdn.com +time=1 +noall +stats<br>
;; global options: +cmd<br>
;; Query time: 45 msec<br>
;; SERVER: 8.8.8.8#53(8.8.8.8)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 525<br>
<br>
;; Query time: 46 msec<br>
;; SERVER: 8.8.8.8#53(8.8.8.8)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 525<br>
<br>
;; Query time: 29 msec<br>
;; SERVER: 199.7.83.42#53(199.7.83.42)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 1168<br>
<br>
;; Query time: 81 msec<br>
;; SERVER: 202.12.27.33#53(202.12.27.33)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 1169<br>
<br>
;; Query time: 212 msec<br>
;; SERVER: 192.48.79.30#53(192.48.79.30)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 946<br>
<br>
;; Query time: 5 msec<br>
;; SERVER: 139.224.142.121#53(139.224.142.121)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 53<br>
<br>
;; Query time: 211 msec<br>
;; SERVER: 192.52.178.30#53(192.52.178.30)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 909<br>
<br>
;; Query time: 8 msec<br>
;; SERVER: 183.192.164.119#53(183.192.164.119)<br>
;; WHEN: Fri Feb 24 16:50:14 CST 2023<br>
;; MSG SIZE  rcvd: 108<br>

</details>

wireshark抓到的包为：

[![pSzgIPg.png](https://s1.ax1x.com/2023/02/24/pSzgIPg.png)](https://imgse.com/i/pSzgIPg)

共16个包。

因为加了&，两个`dig`命令是同一时刻执行的，而且每个DNS请求的响应不全是请求的下一个包。如图响应包和请求包之间就有2个包。我整理了一下：

| 请求包序号 | 响应包序号 | 响应时间 | 第几个dig命令产生的包      | 对应dig命令的输出                                            |
| ---------- | ---------- | -------- | -------------------------- | ------------------------------------------------------------ |
| 23         | 25         | 0.045652 | 不确定，我好像看不出来。。 | ;; Query time: 45 msec<br/>;; SERVER: 8.8.8.8#53(8.8.8.8)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 525 |
| 24         | 26         | 0.046427 | 不确定，我好像看不出来。。 | ;; Query time: 46 msec<br/>;; SERVER: 8.8.8.8#53(8.8.8.8)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 525 |
| 27         | 34         | 0.08078  | 2                          | ;; Query time: 29 msec<br/>;; SERVER: 199.7.83.42#53(199.7.83.42)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 1168 |
| 28         | 29         | 0.028015 | 1                          | ;; Query time: 81 msec<br/>;; SERVER: 202.12.27.33#53(202.12.27.33)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 1169 |
| 32         | 48         | 0.212717 | 1                          | ;; Query time: 212 msec<br/>;; SERVER: 192.48.79.30#53(192.48.79.30)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 946 |
| 35         | 51         | 0.210875 | 2                          | ;; Query time: 5 msec<br/>;; SERVER: 139.224.142.121#53(139.224.142.121)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 53 |
| 49         | 50         | 0.004978 | 1                          | ;; Query time: 211 msec<br/>;; SERVER: 192.52.178.30#53(192.52.178.30)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 909 |
| 52         | 53         | 0.007501 | 2                          | ;; Query time: 8 msec<br/>;; SERVER: 183.192.164.119#53(183.192.164.119)<br/>;; WHEN: Fri Feb 24 16:50:14 CST 2023<br/>;; MSG SIZE  rcvd: 108 |

根据之前所说，每次DNS请求的transaction id都是无序的，在本次抓包结果可以验证。

DNS解析这里是2个命令同时执行的，我理解的就是2个进程分别对应着2个命令的执行，这2个进程分别生成随机的Transaction ID，然后请求，DNS的响应中会有请求的Transaction ID，这2个进程只会处理自己生成的Transaction ID对应的响应，根据响应的结果判断出下一个请求该如何发送。这样发送几次请求后，得到对应的结果。2个进程也不会互相干扰，只会处理跟自己生成的Transaction ID相关的内容。

