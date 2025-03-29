---
title: clickhouse的嵌套类型与Tuple
date: 2023-03-10 11:00:00
categories:
- 数据库
---

下面是关于Nested类型和Tuple类型的SQL用法的例子，两者插入数据的方法不一样，存储方式也不也一样：

```sql
create table test(
    uid Int8 ,
    dns_flag_desc String,
    answer Nested(
        type String,
        info String 
     )
) engine = MergeTree ORDER BY uid ;

## 插入数据
insert into test values(1,'NO ERROR',['A','A'],['1.1.1.1', '2.2.2.2']);
insert into test values(1,'NO ERROR',['A','A','A'],['1.1.1.1', '2.2.2.2','3.3.3.3']);
## 查询数据
select answer.type from test
```

Nested存储到数据库后是存储为了多列，比如这里存储的时候存储为了answer.type和answer.info这两列，删除的时候也要分别删除这两列。

```sql
CREATE TABLE my_table
(
    `id` UInt32,
    `nested_array` Array(Tuple(Int32, String))
)
ENGINE = Memory

## 插入数据
INSERT INTO my_table (id, nested_array)
                VALUES (1, [(10, 'foo'), (20, 'bar')]
INSERT INTO my_table (id, nested_array)
                VALUES (1, [(10, 'foo'), (20, 'bar'), (30, 'bar1')]  
## 查询数据
select nested_array.1 from test # 这里查询的出来的就是第一个列，也就是Int32那一列的数据
```

如果是Array(Tuple(Int32, String))类型的，存储到数据库实际上只存储为了一列，只是这一列是个数组，数组的每个元素是个json对象。

Nested比Tuple更为灵活，可以指定字段名

