---
title: clickhouse入门
date: 2022-7-16 16:04:00
categories:
- 运维
---
## 和其他数据库相比

**它也是一种关系型数据库。**

**传统的关系型数据库是***行式存储*，而clickHouse是**列式存储**。

**关于clickHouse和mysql的对比，但空间唯独上可以抽象为行（横轴）列（纵轴），行式存储位于一行的数据总是被物理存储在一起。**

**列式存储**的优势：更适合OLAP（是仓库型数据库，**主要是读取数据，做复杂数据分析，侧重技术决策支持**，提供直观简单的结果。和OLAP相对的是OLTP，是传统的关系型数据库，主要操作增删改查，**强调事务一致性**）

**缺点：**

1. **不支持事务**。不要把clickHouse直接当做像mysql这样的关系型数据库。clickHouse本身的定位就是用于联机分析(OLAP)的列式数据库管理系统(DBMS)。
2. **关于实时性**。缺少高频率，低延迟的修改或删除已存在数据的能力。仅能用于批量删除或修改数据，但这符合 [GDPR](https://gdpr-info.eu/)。就像elasticsearch搜索引擎，它也是近实时的。如果对实时性有钱强烈要求的，应该避免使用clickHouse，或者通过业务上的设计，合理避免实时性问题。
3. **稀疏索引使得ClickHouse不适合通过其键检索单行的点查询。**

## 启动服务

**使用 docker**

```
docker run -d --name clickhouse-server --ulimit nofile=262144:262144 -p 9000:9000 yandex/clickhouse-server:1.1
```

**或者读取自定义配置运行：**

**注意，需要在$PWD/etc/clickhouse-server/config.xml指定时区** `<timezone>Asia/Shanghai</timezone>`

```
docker run -d --name some-clickhouse-server --ulimit nofile=262144:262144 -p 8123:8123 -p 9000:9000 --volume=$PWD/some_clickhouse_database:/var/lib/clickhouse --volume=$PWD/etc/clickhouse-server:/etc/clickhouse-server yandex/clickhouse-server:latest
```

> **ps：对于clickhouse-server:1.1（但是实际上运行时用的不是1.1），我打开了<listen_host>::</listen_host>，这个配置是限制来源主机的请求，<listen_host>::</listen_host>的意思是允许IP4和IP6源主机远程访问。具体可以参见config.xml的注释。**

**启动客户端：**

```
docker run -it --rm --link some-clickhouse-server:clickhouse-server yandex/clickhouse-client --host clickhouse-server
```

> **补充的一些命令：**
>
> ```
> CREATE DATABASE db_1;
> show databases;
> CREATE TABLE test02( id UInt16,col1 String,col2 String,create_date date ) ENGINE = MergeTree(create_date, (id), 8192);
> insert into test02 values(1, 'tank','tank',  '2021-08-19 14:21:30');  
> ```
>
> ```
> docker exec -i -t xxx /bin/bash
> ```

## 语法

**与常用的SQL一致，需要注意的是创建表的时候需要指定Engine。**

```
create (temporary) table (if not exists) test.m1 (
 id UInt16
,name String
) ENGINE = Memory
;
```

## 数据类型

**整型：UInt8,UInt16,UInt32,UInt64,Int8,Int16,Int32,Int64**

**枚举类型：Enum8,Enum16，存储为Int8或Int16的一组枚举字符串值。Enum8（'hello'= 1，'world'= 2）， 这个数据类型有两个值  'hello'和'world'。**

**字符串(String、FixedString 和 UUID，FixedString 固定了长度，UUID为32位**

**数组类型：Array(T)表示T类型的数组**

**元组：Tuple，由多个元素组成，允许不同类型**

**嵌套（Nested(Name1 Type1, Name2 Type2, ...)），MergeTree 引擎中不支持存储这样的列。不能对整个嵌套数据结构执行 SELECT。只能明确列出属于它一部分列**

**定点数 Decimal32、Decimal64 和Decimal128，Decimal(P, S)：P代表精度，决定总位数（整数部分+小数部分）**

**缺失值（Nullable(TypeName)），**`Nullable(Int8)` 类型的列可以存储 `Int8` 类型值，而没有值的行将存储 `NULL`。使用 Nullable 几乎总是对性能产生负面影响。例子：

```
CREATE TABLE t_null(x Int8, y Nullable(Int8)) ENGINE TinyLog
INSERT INTO t_null VALUES (1, NULL)
```

### 物化列

**指定 MATERIALIZED 表达式，即将一个列作为** `物化列`处理了，这意味着这个列的值不能从 `insert` 语句获取，只能是自己计算出来的。同时，物化列也不会出现在 `select *` 的结果中：

```
drop table if exists test.m2;
create table test.m2 (
 a MATERIALIZED (b+1)
,b UInt16
) ENGINE = Memory;
insert into test.m2 (b) values (1);
select * from test.m2;
select a, b from test.m2; --这个可以查出来
```

### 表达式列

**ALIAS 表达式列某方面跟物化列相同，就是它的值不能从 insert 语句获取。不同的是， 物化列 是会真正保存数据（这样查询时不需要再计算），**

**而表达式列不会保存数据（这样查询时总是需要计算），只是在查询时返回表达式的结果。**

```
create table test.m3 (a ALIAS (b+1), b UInt16) ENGINE = Memory;
insert into test.m3(b) values (1);
select * from test.m3;
select a, b from test.m3;
```

## Engine

### 数据库引擎

**Atomic**：它支持非阻塞 DROP 和 RENAME TABLE 查询以及原子 EXCHANGE TABLES t1 AND t2 查询。

**Mysql** MySQL引擎用于将远程的MySQL服务器中的表映射到ClickHouse中，并允许您对表进行 `INSERT`和 `SELECT`查询，以方便您在ClickHouse与MySQL之间进行数据交换。创建数据库时需要指定原始Mysql数据源。

`MySQL`数据库引擎会将对其的查询转换为MySQL语法并发送到MySQL服务器中，因此您可以执行诸如 `SHOW TABLES`或 `SHOW CREATE TABLE`之类的操作。

**示例：**

```
CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster]
ENGINE = MySQL('host:port', ['database' | database], 'user', 'password')

MySQL数据库引擎参数
host:port — 链接的MySQL地址。
database — 链接的MySQL数据库。
user — 链接的MySQL用户。
password — 链接的MySQL用户密码。
```

**Lazy**

**在距最近一次访问间隔** `expiration_time_in_seconds`时间段内，将表保存在内存中，仅适用于 Log引擎表

*由于针对这类表的访问间隔较长，对保存大量小的 *Log引擎表进行了优化。**

```
CREATE DATABASE testlazy ENGINE = Lazy(expiration_time_in_seconds);
```

### 表引擎

#### Log系列

**主要用于快速写入小表（1百万行左右），然后全读出来的场景。当需要快速写入和整体读的时候，最有效。**

**Log系列表引擎的共性是：**

1. **数据顺序append写到磁盘**
2. **不支持delete和update**
3. **不支持索引**
4. **不支持原子写**
5. **Insert的操作会阻塞select操作**

**主要的特点是：**

1. **数据存储磁盘**
2. **写入时追加数据到文件末尾**
3. **不支持突变的操作**
4. **不支持索引，所以select效率比较低**
5. **非原子性写入数据**

**其中的tinylog一般用于测试，存储少量数据的小表：**

```
CREATE TABLE t_tinylog(
id Int64,
  name String
)
ENGINE = TinyLog;
```

#### Integration系列

**该系统表引擎主要用于对接外部数据源**

**Kafka：可以将kafka Topic中的数据直接导入到clickhouse**

**Mysql：将Mysql作为存储引擎，直接可以在Clickhouse中对mysql表进行select等操作**

**HDFS：直接读取HDFS的特定格式的数据文件**

#### Specal系列

**Memory引擎，数据以未压缩的原始形式存在内存中，服务器重启数据就会消失。读写操作不会相互阻塞，不支持索引。简单查询下有非常高的性能表现。一般仅用来测试，适用于非常高的性能，同事数据量不大的场景。**

**Merge引擎本身不存储数据，但同时从任意多个其他的表中读取数据。读取时自动并行，支持写入。**

**Distributed引擎，本身不存储数据，但是可以从多个服务器进行分布式查询。读是自动并行的**

**关于整合：**

1. **Merge引擎：在同一个服务器上的，多个相同结构的物理表，可以被整合成一张大的逻辑表，这张表的数据包含了物理表中的所有数据**
2. **Distributed：在不同的server上，多个相同结构的物理表，可以被整合成一张大的逻辑表，这张逻辑表的数据，就是包含了物理表的所有数据**

#### MergeTree

**绝大数场景会使用MergeTree**

**只有MergeTree系列的表引擎才支持**[主键](https://so.csdn.net/so/search?q=%E4%B8%BB%E9%94%AE&spm=1001.2101.3001.7020)索引，数据分区，数据副本，数据采样这些特性，只有此系列的表引擎才支持ALTER操作。

## 聚合

**click house支持聚合操作，有一些标准聚合函数:**8

**count，min，max等**

**ClickHouse 特有的聚合函数:**

**anyHeavy，anyLast，argMin等**

**参考**[官方文档](https://clickhouse.com/docs/zh/sql-reference/aggregate-functions/parametric-functions)
