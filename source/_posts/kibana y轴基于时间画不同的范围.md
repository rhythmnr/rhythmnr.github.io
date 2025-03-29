---
title: kibana y轴基于时间画不同的范围
date: 2022-11-05 10:26:00
tags:
- 原创
categories:
- 运维
---

最终效果

![WX20221028-150039@2x](https://tva2.sinaimg.cn/mw690/006gLprLgy1h7l0i95sjtj323c0p8qfz.jpg)

画图步骤

选择TSVB

![WX20221028-150212@2x](https://tvax4.sinaimg.cn/mw690/006gLprLgy1h7l0j9ssh5j318s0wq7bi.jpg)

选择Time Series，配置如下：

![WX20221028-150313@2x](https://tva4.sinaimg.cn/mw690/006gLprLgy1h7l0kbba69j323w10en7q.jpg)

## y轴画不同的范围

选择Vertical Bar的Visualization，然后根据不同范围统计的范围限制在filter中，可以添加任意个filter。

![WX20221028-160632@2x](https://tvax4.sinaimg.cn/mw690/006gLprLgy1h7l2ekp52dj323811iwpd.jpg)
