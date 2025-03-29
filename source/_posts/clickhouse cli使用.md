---
title: clickhouse cli使用
date: 2023-02-21 11:00:00
categories:
- 数据库
---


参考[官方文档](https://clickhouse.com/docs/zh/interfaces/cli/)

比如执行下面的语句，就相当于登录上了clickhouse服务器并执行了SQL语句

```sql
clickhouse-client --host="x" --user="x" --database="x" --password="x" --query="select count() from x where create_time < date_add(DAY, -7, date_trunc('day', now()));";
```

