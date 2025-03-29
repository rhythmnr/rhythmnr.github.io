---
title: 如何通过wireshark抓VPN的数据包
date: 2023-1-23 00:00:00
categories:
- 计算机网络
---
当VPN连接建立后，如果你使用ping命令进行网络测试，那么ping的流量将通过VPN网络接口进行传输。这是因为VPN连接会创建一个虚拟网络接口，所有的网络流量都将通过该接口转发到VPN服务器。这样做可以保证网络流量的安全性和隐私性。

这里举个例子： 假设你的网络接口为eth0，你在连接VPN后，系统会创建一个新的VPN接口，如tun0，在这种情况下，ping的流量将通过tun0接口进行传输。

注意:

- 这只是一般情况，具体情况还要根据你使用的VPN客户端和VPN服务器的配置而定。
- 你可以使用命令“ipconfig /all”(Windows) 或 “ifconfig”(Linux)来查看你的网络接口。

比如我未连接VPN前，通过ifconfig查看了所有网络接口。连接了VPN后，通过ifconfig查看了所有网络接口，发现比之前多了一个utun3，那么Wireshark抓包的时候抓这个utun3接口就可以抓到所有VPN的流量。