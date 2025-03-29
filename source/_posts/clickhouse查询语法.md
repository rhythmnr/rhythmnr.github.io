---
title: clickhouse查询语法
date: 2022-11-25 11:04:00
categories:
- 数据库
---
## group by ID

```
select  src_ip, dst_ip, SUM(pack_size) AS value, toStartOfInterval(create_time, INTERVAL 15 minute) AS time
from tableA 
where create_time >= '2022-11-17 16:15:00' and  create_time< '2022-11-17 17:20:00'
group by 4, 1, 2
order by 4, 3 desc
```

clickhouse的group by和order by可以使用字母，字母代表select后面的第几个元素，比如上面的查询效果等同于

```
select  src_ip, dst_ip, SUM(pack_size) AS value, toStartOfInterval(create_time, INTERVAL 15 minute) AS time
from tableA 
where create_time >= '2022-11-17 16:15:00' and  create_time< '2022-11-17 17:20:00'
group by time, src_ip, dst_ip
order by time, value desc
```

## toStartOfInterval

获得指定时间根据interval划分后的时间，相当于指定时间除以interval后取整然后乘以interval，比如15点12分和10分钟interval后的结果就是15点10分。有一些场景，比如统计每10分钟的平均数量，那么一小时就 有6个平均数量的值的时候，使用toStartOfInterval对时间进行处理就很有用，可以判断这个时间应该算到第几个十分钟里去。比如这里kibana的Lens图像是每10分钟为单位的，对应到CK的查询可以用toStartOfInterval实现聚合，参考上面的SQL语句。

[![zYiWdO.png](https://s1.ax1x.com/2022/11/25/zYiWdO.png)](https://imgse.com/i/zYiWdO)

## quantileExact

计算数字序列的[分位数](https://en.wikipedia.org/wiki/Quantile)。

语法：`quantileExact(level)(expr)`

比如 `SELECT quantileExact(number) FROM numbers(10)`返回的 `numbers(10)`默认的分位数5

## round

```
SELECT round(1.1233456789,6) 
```

返回了1.123346，就是取1.1233456789到小数点后的6位，不会做四舍五入

bewtween a and b的范围是[a,b]

涉及到时间查询，一般写左闭右开，比如t<end_time and t>=start_time

## 一个复杂查询的示例与解释

### 源语句

这是工作解决某个问题对应的查询语句

```
SELECT 
  u.ct, 
  untuple(
    arrayJoin(
      arraySlice(
        arraySort(
          (x, y)->- y, 
          arrayMap(
            (x, y, z) -> (x, y, z), 
            groupArray(u.src_ip), 
            groupArray(u.dst_ip), 
            groupArray(u.p)
          ), 
          groupArray(u.p)
        ), 
        1, 
        25
      )
    )
  ) AS res 
FROM 
  (
    SELECT 
      toStartOfInterval(create_time, INTERVAL 1 minute) AS ct, 
      src_ip, 
      dst_ip, 
      SUM(pack_size) AS p 
    FROM 
      `tableA` 
    WHERE 
      create_time < FROM_UNIXTIME(1668700800) 
      AND create_time >= FROM_UNIXTIME(1668672000) 
    GROUP BY 
      src_ip, 
      dst_ip, 
      ct
  ) as u 
GROUP BY 
  `ct` 
ORDER BY 
  ct
```

相关的函数

**untuple**

将元组拆分为列，umtuple(x) as res得到的列分别为res.1 res.2

例子:

```
SELECT
    (1, 'a').1,
    (1, 'a').2

/*
┌─tupleElement(tuple(1, 'a'), 1)─┬─tupleElement(tuple(1, 'a'), 2)─┐
│                              1 │ a                              │
└────────────────────────────────┴────────────────────────────────┘
*/
```

**arrayJoin**

用于将结果中的数组展开，来自官方的例子

```
SELECT arrayJoin([1, 2, 3] AS src) AS dst, 'Hello', src
┌─dst─┬─\'Hello\'─┬─src─────┐
│   1 │ Hello     │ [1,2,3] │
│   2 │ Hello     │ [1,2,3] │
│   3 │ Hello     │ [1,2,3] │
└─────┴───────────┴─────────┘
```

**arraySlice**

返回一个子数组，包含从指定位置的指定长度的元素。arraySlice[arr,1,n]返回arr[1:n]共n个元素的数组

**groupArray**

创建数组，根据某列

**arrayMap**

根据多个数组，创建数组间的映射关系返回映射结果。这里的根据3个数组返回这三个数组拼成的元组。

**arraySort**

返回升序排序 `arr1`的结果。如果指定了 `func`函数，则排序顺序由 `func`的结果决定。

示例:

```sql
SELECT arraySort((x, y)-> y,['hello','world'],[2,1]);
```

```text
┌─res────────────────┐
│ ['world', 'hello'] │
└────────────────────┘
```

**toTypeName(x)可以获得数据类型**

`SELECT toTypeName(arrayMap((x, y) -> (x, y), [1, 2, 3], [4, 5, 6]))`

### 简化

过了很长时间后，在看将clickhouse的书时发现上面的查询语句可以用一种更简单的写法实现：

```go
 SELECT 
      toStartOfInterval(create_time, INTERVAL 1 minute) AS ct, 
      src_ip, 
      dst_ip, 
      SUM(pack_size) AS p 
    FROM 
      flora.gopacket
    WHERE 
        create_time < FROM_UNIXTIME(1676455200) 
        AND create_time >= FROM_UNIXTIME(1676433600)
    GROUP BY 
      src_ip, 
      dst_ip, 
      ct
     ORDER BY ct, p DESC 
     LIMIT 25 BY ct
```

这个功能就是相当于在排序完成之后，对于每个ct，取前25个值。因为之前已经ORDER BY p DESC了，所以可以获得前25个值最大的p。

参考

[in-clickhouse-how-to-parse-date-datetime-in-a-given-format](https://stackoverflow.com/questions/70740482/in-clickhouse-how-to-parse-date-datetime-in-a-given-format)

[tostartofhour](https://clickhouse.com/docs/en/sql-reference/functions/date-time-functions/#tostartofhour)
