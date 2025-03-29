---
title: clickhouse查询实践与进阶
date: 2022-12-13 11:00:00
categories:
- 数据库
---
table里记录了一些流量详情，流量包括源IP src_ip，目标IP dst_ip，数据包大小pack_size，创建时间create_time

下面是一些要统计的内容

## 一个ip和多少个ip通信了

查询每个src_ip对应有多少个dst_ip，每个dst_ip对应多少个src_ip，也就是一个ip和多少个ip通信了，SQL语法如下：主要就是union all的两边。

```sql
select p, count(distinct(q)) from (
select r.1 as p, r.2 as q from (
select distinct(tuple(src_ip,dst_ip)) as r
from table
where create_time >= '2022-12-03 10:00:00' and create_time < '2022-12-03 10:10:00'
)
union all 
select r.2 as p, r.1 as q from (
select distinct(tuple(src_ip,dst_ip)) as r
from table
where create_time >= '2022-12-03 10:00:00' and create_time < '2022-12-03 10:10:00'
)
)
group by p
```

## 数据包大小分布

要计算pack_size<100的数据包有多少个，100<=pack_size<200的数据包有多少个，200<=pack_size<300的数据包有多少个等等

```sql
select  multiIf(pack_size <100, 1, pack_size <200, 2, pack_size <300, 3, pack_size <400, 4, pack_size <500, 5, pack_size <600, 6, pack_size <700, 7, 0),pack_size from table
where create_time >= '2022-12-03 10:00:00' and create_time < '2022-12-03 10:10:00'
limit 10
```

这个是运行结果

```shell
3	274
3	274
2	138
3	274
1	60
7	638
1	60
2	122
2	138
2	198
```

改成这样也可以：

```sql
select  multiIf(pack_size <100, 1, pack_size <200, 2, pack_size <300, 3, pack_size <400, 4, pack_size <500, 5, pack_size <600, 6, pack_size <700, 7, 0),count(*) from table
where create_time >= '2022-12-03 10:00:00' and create_time < '2022-12-03 10:10:00'
group by 1
```

主要就是通过[multiIf](https://clickhouse.com/docs/zh/sql-reference/functions/conditional-functions)实现的。

> 下面的内容来自clickhouse官方文档：
>
> multiIf允许您在查询中更紧凑地编写[CASE](https://clickhouse.com/docs/zh/sql-reference/operators/#operator_case)运算符。
>
> **语法**
>
> ```sql
> multiIf(cond_1, then_1, cond_2, then_2, ..., else)
> ```
>
> 您可以使用[short_circuit_function_evaluation](https://clickhouse.com/docs/zh/operations/settings/settings#short-circuit-function-evaluation) 设置，根据短路方案计算 `multiIf` 函数。如果启用此设置，则 `then_i` 表达式仅在 `((NOT cond_1) AND (NOT cond_2) AND ... AND (NOT cond_{i-1}) AND cond_i)` 为真，`cond_i `将仅对 `((NOT cond_1) AND (NOT cond_2) AND ... AND (NOT cond_{i-1}))` 为真的行进行执行。例如，执行查询“SELECT multiIf(number = 2, intDiv(1, number), number = 5) FROM numbers(10)”时不会抛出除以零的异常。
>
> **参数:**
>
> - `cond_N` — 函数返回`then_N`的条件。
> - `then_N` — 执行时函数的结果。
> - `else` — 如果没有满足任何条件，则为函数的结果。
>
> 该函数接受`2N + 1`参数。

