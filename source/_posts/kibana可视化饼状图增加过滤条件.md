---
title: kibana可视化饼状图增加过滤条件
date: 2022-9-18 11:00:00
tags:
- 原创
categories:
- 运维
---
前提：在kibana可视化中，需要选择饼图呈现统计结果。

要求：饼图只统计http_type:response的数据。

修改前只有这一个Buckets：

![WX20220916-161320@2x](https://tvax1.sinaimg.cn/large/006gLprLgy1h68iklskc7j325c12knan.jpg)

需要在这个Bucket前面增加一个Split chart，Split chart内增加过滤条件。

修改后：

![WX20220916-161533@2x](https://tva3.sinaimg.cn/large/006gLprLgy1h68immmfl6j325m12i7h3.jpg)

