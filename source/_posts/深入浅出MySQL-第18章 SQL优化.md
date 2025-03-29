---
title: 深入浅出MySQL-第18章 SQL优化
date: 2019-6-24 00:00:00
tags:
- 读书
categories:
- 数据库
---

18.1 优化SQL的一般步骤
18.1.1 通过show status了解各种SQL的执行效率

1. show session status 显示当前连接的状态
2. show global status 显示从数据库启动开始到现在连接的状态
3. show status like 'Com_%s' 显示类似某种状态的信息
18.1.2 定位执行效率低的SQL语句
4. 定位执行效率低的SQL语句:
show processlist 可以查看当前MySQL正在进行的进程
18.1.3 通过EXPLAIN分析低效的SQL执行计划
1. 在sql语句前加上explain可以看到执行的时候的各个信息
2. 可以通过 explain extended  加上show warnings，查看SQL被执行前优化器做了哪些修改，通过filtered字段查看。
18.1.4 通过show profile分析SQL
1. 查看是否支持 profile
show @@have_profilling
2. 查看状态
show profiles
3. 查看查询的状态，查看每个线程的时间
show profile for query query_id
show profile cpu for query query_id 查看cpu的时间
4. mysql设置常量
set @name:=value;
5. 改变表的引擎
alter table tablename engine = ***
18.1.5 通过trace分析优化器如何选择执行计划
1. 打开trace，设置格式为json
set OPTIMIZER_TRACE = 'enabled=on',END_MARKERS_IN_JSON = on;
2. 设置trace能够使用的最大内存
set OPTIMIZER_TRACE_MAX_MEM_SIZE = 1000000
3. 执行完成sql语句，查看INFORMATION_SCHEMA.OPTIMIZER_TRACE，可以看见mysql是如何执行sql语句的。
18.1.6 确定问题并采取对应的优化措施
在一些列上建立索引可以极大提高查询的效率，可以通过explain命令查看受到影响的行数。
18.2 索引问题
18.2.1 索引的存储分类
目前mysql提供四种索引：
B-Tree索引
HASH索引
R-Tree索引（空间索引）
Full-Text索引（全文索引）
可以对字段前几个进行索引
18.2.2 Mysql如何使用索引
1. 三种存储引擎都支持B-Tree索引
2. B-Tree可以进行全关键字，关键字范围和关键字前缀查询。
<https://www.jianshu.com/p/486a514b0ded>

3. mysql能够使用引擎的情况
匹配全值（Match the full value)
匹配值的范围查询(Match a range of values)
匹配最左前缀(Match a leftmost prefix)
仅仅对索引进行查询(Index only query)
 匹配列前缀(Match a column prefix)
 索引部分精确匹配而其他部分范围匹配(Match one part exactly and match a range on another part)
使用列名为空
可以根据explain命令后显示的key判断使用了什么索引
type为range说明使用范围查询
extra为using where说明除了利用索引加速访问外，还根据索引回表查询数据
extra为using index说明只需要查询索引就获得了需要的内容
4. 存在索引但是不能使用索引的典型场景
1. 例子
select * from (select actor_id from actor where last_name like '%NI%')a ,actor b where a.actor_id = b.actor_id
