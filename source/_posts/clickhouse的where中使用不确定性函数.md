---
title: clickhouse的where中使用不确定性函数
date: 2023-02-21 11:00:00
categories:
- 数据库
---
## 需求

要实现一个脚本，删除clickhouse七天前的数据

## SQL

找到了date_trunc函数，参考了 https://clickhouse.com/docs/en/sql-reference/functions/date-time-functions/

官方用法，比如`SELECT now(), date_trunc('hour', now());`

```sql
date_trunc(unit, value[, timezone])
```

Alias: `dateTrunc`.

**Arguments**

- `unit` — The type of interval to truncate the result. [String Literal](https://clickhouse.com/docs/en/sql-reference/syntax#syntax-string-literal). Possible values:
  - `second`
  - `minute`
  - `hour`
  - `day`
  - `week`
  - `month`
  - `quarter`
  - `year`
- `value` — Date and time. [DateTime](https://clickhouse.com/docs/en/sql-reference/data-types/datetime) or [DateTime64](https://clickhouse.com/docs/en/sql-reference/data-types/datetime64).
- `timezone` — [Timezone name](https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings#server_configuration_parameters-timezone) for the returned value (optional). If not specified, the function uses the timezone of the `value` parameter. [String](https://clickhouse.com/docs/en/sql-reference/data-types/string).

执行：

`ALTER table t ON CLUSTER c DELETE WHERE create_time < date_add(DAY, -7, date_trunc('day', now())))`

报错了：ALTER UPDATE/ALTER DELETE statements must use only deterministic functions.

报错原因：因为clickhouse在where中使用不确定性函数，`now()` 函数在 ClickHouse 中也是不确定性函数。"date_add" 和 "date_trunc" 不是确定性函数。where后面不可以放上不确定性函数。

## 通过shell脚本实现

```shell
#!/bin/bash
# 该脚本的功能为：删除clickhouse里的七天前的数据

BEGIN=$(date +%s --date='-7 day')
HOST=x
DATABASE=x
USERNAME=x
PASSWORD=x
TABLE=x
CLUSTER=x

clickhouse-client \
--host="$HOST" \
--user="$USERNAME" \
--database="$DATABASE" \
--password="$PASSWORD" \
--query="ALTER table $TABLE ON CLUSTER $CLUSTER DELETE WHERE create_time < FROM_UNIXTIME($BEGIN);"
```



