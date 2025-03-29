---
title: kibana的Lens计算原理
date: 2022-11-01 20:00:00
tags:
- 原创
categories:
- 运维
---

使用kibana的Lens功能画的图，每个采样点的值是如何计算的呢？

比如这个采样点的值是如何计算出来的？

![WX20221105-103709@2x](https://tvax2.sinaimg.cn/large/006gLprLgy1h7u1uy1ywwj31ic0rkdob.jpg)

![WX20221104-152741@2x](https://tvax1.sinaimg.cn/large/006gLprLgy1h7t4ly6tg0j325k12o17v.jpg)

可以看到，这里是每30秒一个采样点。后台配置如下：

![WX20221104-153047@2x](https://tva2.sinaimg.cn/large/006gLprLgy1h7t4p5yvnej31qg0vctnw.jpg)

经过一番研究与测试，过程在这里就不赘述了因为挺繁琐，最终总结得到的计算方法是：

14:45:30秒的数据的计算方法是：

14:45:30到14:45:59（不包括14:50:00，即左闭右开）的所有pack_size求和，然后和处以时间间隔即30，就可以得到对应的图上的值，也就是后台配置的per second的值。

至于采样开始的值，会使用一个基于时间间隔的时间点，比如这里指定的开始时间是37分37秒，最终得到的图的第一个采样点的时间是38秒。

![WX20221104-153643@2x](https://tva3.sinaimg.cn/large/006gLprLgy1h7t4vbrf8lj31s00v0ao5.jpg)

最后一个采样点不大于指定的结束时间：

![WX20221104-153913@2x](https://tvax1.sinaimg.cn/large/006gLprLgy1h7t4xy12n9j31s60vcqll.jpg)
