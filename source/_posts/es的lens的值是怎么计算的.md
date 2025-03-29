---
title: es的lens各个点的值是怎么计算的
date: 2022-12-20 10:00:00
categories:
- 运维
---
[![z0aMR0.png](https://s1.ax1x.com/2022/12/01/z0aMR0.png)](https://imgse.com/i/z0aMR0)

主要讨论关于图上的值 6974378是如何计算的

通过实验，12:00这一个时刻发现这个值在ES中的计算方法对应的SQL可以写为：

```
select SUM(pack_size) FROM table where create_time < FROM_UNIXTIME(1669867260) AND create_time >= FROM_UNIXTIME(1669867200)
```

1669867260对应的时间戳是12:00，1669867200对应的时间戳是12:01

所以显示的数组是每分钟为一个单位计算的和，后台配置的是SUM

所以图上配置的图表写字节/每秒不合适，应该写字节/每分钟，和下面横坐标的per minute对应

