---
title: 深入浅出MySQL-第17章 MySQL分区
date: 2019-5-1 00:00:00
tags:
- 读书
categories:
- 数据库
---

分区是指数据库将一个表分为更小，更容易管理的部分。逻辑上仍然只有一个表。
每个分区都是独立的对象，可以独自处理，也可以作为表的一部分处理。分区对于应用来说是透明的，不影响应用的业务逻辑。
17.1 分区概述

1. 分区引入了分区键（partition key）的概念，分区键根据某个区间值（或者范围值），特定列表或者HASH函数执行数据的聚集，让数据根据规则分布在不同的分区中，让一个大对象变成一些小对象。
2. 查看是否可以分区:  show variables like '%partition%'
3. 对同一个分区表的所有分区都必须使用同一个存储引擎
同一个数据库中不同分区表可以使用不同的存储引擎
4. 每个分区表也要设置存储引擎
create table temp (id int,birthday DATE)
engine = innodb #分区表存储引擎为InnoDB，必须在create table 语句之后才可以出现
paritition by hash(month(birthday)) # 数据聚集方法
partitions 6; # 指定6个分区
17.2 分区类型
1. 几种分区类型
RANGE 分区:给定一个连续区间范围，把数据分配到不同的分区
LIST 分区：类似于RANGE分区，区别是LIST分区是基于枚举出的值列表分区，RANGE是基于给定的连续区间范围分区
HASH 分区：基于给定的分区个数，将数据分配到不同的分区
KEY 分区：类似于HASH分区
如果一个表有主键的话，那么分区应该基于该主键。如果没有的话，就随便基于哪个字段。
17.2.1 Range分区
1. 分区创建语句
create table tem(
id int not null,
ename varchar(30),
hired DATE NOT NULL DEFAULT '1970-01-01',
job varchar(30) NOT NULL,
stored_id INT NOT NULL
)
partition by range(stored_id){
partition p0 values less than (10),
partition p1 values less than (20),
partition p2 values less than (30),
);
就像java 的switch case 语句。
也可以指定当小于某个值（maxvalue）的时候就到指定的分区
alter table tem add paritition (partition p3 values less than maxvalue)
2. 删除分区
alter table emp drop paritition p0
17.2.2 List分区
1. List分区与Range分区类似，List分区是从属一个枚举列表的值的集合，range是从属一个连续区间值的集合。
2. 语法
create table tem(
id int not null,
ename varchar(30),
hired DATE NOT NULL DEFAULT '1970-01-01',
job varchar(30) NOT NULL,
stored_id INT NOT NULL
)
partition by list(id){
partition p0 values in(3,5),
partition p1 values in(4),  # in 里面的东西也可以不是整数
partition p2 values in(8),
);
如果插入的值不在范围内，那么会报错，毕竟没有什么 values less than 等等的写法。
17.2.3 Columns分区
1. columns分区可以细分为 RANGE Columns 分区和 LIST Columns分区， RANGE Columns 分区和 LIST Columns分区都支持整数，日期时间，字符串三种数据类型。
2. columns分区支持多列分区，是基于元组的比较
create table tem(
id int not null,
ename varchar(30),
hired DATE NOT NULL DEFAULT '1970-01-01',
job varchar(30) NOT NULL,
stored_id INT NOT NULL
)
partition by columns(id,stored_id){
partition p01 values less than (0,10),
partition p02 values less than (10,10),  # in 里面的东西也可以不是整数
partition p03 values less than (10,maxvalue),
partition p03 values less than (maxvalue,maxvalue)
);
17.2.4 Hash分区
1. Hash 分区主要用来分散热点读，确保数据在预先确定个数的分区中尽可能平均分布。
2. 有两种Hash算法：常规Hash分区和线性Hash分区(LINEAR HASH分区），常规HASH使用取模算法，线性HASH分区使用一个线性的2的幂的运算法则。
常规Hash分区算法：
使用 N=MOD(expr,num) N为几就去第几个分区
create table tem(
id int not null,
ename varchar(30),
hired DATE NOT NULL DEFAULT '1970-01-01',
job varchar(30) NOT NULL,
stored_id INT NOT NULL
)
partition by hash (stored_id) partitions 4; # 有4个分区
线性Hash分区算法：
create table tem(
id int not null,
ename varchar(30),
hired DATE NOT NULL DEFAULT '1970-01-01',
job varchar(30) NOT NULL,
stored_id INT NOT NULL
)
partition by linear hash (stored_id) partitions 4; # 有4个分区
（TODO: 线性Hash算法的计算看不懂）
17.2.5 Key分区
hash分区允许用户使用自定义的表达式，但是Key分区不允许用户使用自定义的表达式，需要使用MySQL提供的HASH函数。
同时Hash分区只支持整数分区，而Key分区支持使用除了BLOB or Text类型外的其他类型的列作为分区键。
create table tem(
id int not null,
ename varchar(30),
hired DATE NOT NULL DEFAULT '1970-01-01',
job varchar(30) NOT NULL,
stored_id INT NOT NULL
)
partition by key  (stored_id) partitions 4; # 有4个分区
如果没有指定分区键，那么会默认使用主键，没有主键会选择非空唯一键。
17.2.6 子分区
1.子分区(subpartitioning)是分区表对每个分区的再次分割，又称为复合分区（composite partitioning）。
支持对range或者list分区在进行子分区，子分区也是list或者range分区。
例子：
create table ts (id int,purchased date)
partition by range(year(purchased))
subpartition by hash(to_year(purchased))
subpartitions 2
(
partition p0 values less than (1900),
partition p0 values less than (2000),
partition p0 values less than maxvalue
);
17.2.7 MySQL分区处理NULL值的方式
1. range分区中，null会被当做最小值处理
list分区中，null必须出现在枚举值列表中
hash/key分区中，null会被当做零值来处理
2. 例子
创建分区：
create table tb_range(
id int,
name varchar(5)
)
partition by range(id){
partition p0 values less than (-6)，
partition p0 values less than (1)，
partition p0 values less than maxvalue  #这个maxvalue是mysql系统自定义的，不是用户自定义的，所以直接写就可以了。
);
查看分区：
select
partition_name part,
partition_expression expr,
partition_description descr,
table rows
from
information_schema.partitions
where
table_schema = schema()
and table_name = 'tb_range';
17.3 分区管理
17.3.1 RANGE&LIST分区管理
1.删除分区：alter table drop partition partitionname
2.删除完分区之后可以看一下： show create table tablename
3.增加range分区，只能从最大端增加： alter table tablename add partition(partition  partitionname values less than (xxx))
也就是partitionname必须是连续的，比如p0,p1,p2,然后如果要增加的话就写p3
4.增加list分区：也只能从最大端增加： alter table tablename add partition(partition  partitionname values in (xxx))
5.修改分区：alter table reorganize partition into
重新定义range分区的时候，只能重新定义相邻的分区，不能跳过分区定义。且新分区必须和旧分区覆盖相同的区间。更不能改变分区的类型。
拆分range分区：
alter table tablename reorganize partition p3 into (
partition p2 values less than (2005),
partition p3 values less than (2010));
对于list分区，如果想要在里面加上枚举值，就需要先加上一个分区，然后再修改。具体可以见书上的例子。
17.3.1 HASH&KEY分区管理
不可以执行像上面这样的直接删除的方式，删除分区。可以通过alter table coalesce partition 语句来合并hash分区或者key分区。
alter table tablename coalesce partition 2; # 可以用来减少分区的数量
不可以增加分区，如果要增加分区的话，alter table add pattition 数量
就是在原有分区的数量上加上
