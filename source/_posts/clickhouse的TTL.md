---
title: clickhouse的TTL
date: 2023-03-10 11:10:00
categories:
- 数据库
---

redis有TTL功能，可以实现指定时间段后自动删除key，clickhouse也有类似的功能，可以指定在某个时间段后自动删除数据，但是删除不是即使的，clickhouse会间隔某个周期（一般是4小时）查看数据库是否有要删除的数据，再对要删除的数据进行删除操作。具体可以参考[官方文档](https://clickhouse.com/docs/en/guides/developer/ttl/)

新增或者删除TTL都可以用modify语句

`ALTER TABLE example1 MODIFY TTL timestamp + INTERVAL 1 HOUR;`

这个表里每一行数据的过期时间为timestamp加上一个小时，过了这个时间一般这行数据就会被自动删除。如果没有自动删除，可以执行`OPTIMIZE TABLE example1 FINAL`手动触发clickhouse的删除数据操作，验证结果。