---
title: 深入浅出MySQL-第2章 SQL基础
date: 2019-1-11 00:00:00
tags:
- 读书
categories:
- 数据库
---

1. SQL语句的分类
DDL：数据定义语言，关键词包括create，drop,alter
DML:数据操纵语言，关键词包括 insert,delete,update,select
DCL:数据控制语言，控制不同的数据段直接许可和访问的级别的语句。grant,revoke
所有的DDL和DML（不包括select），执行成功之后都显示"Query OK"
2.一些命令（DDL与DML)
create database databasename
use databasename
show tables
drop database databasename
alter table tablename modify [column] column_definition [first | after col_name]
字段修改名字： alter table change [column] old_column_name new_column_name new_column_type
查询不重复的记录使用distinct    select distinct col_name from tablename
mysql排序默认降序
聚合操作：group by ,having
内连接：选出两张表相同的部分
外连接：会选出其他不匹配的部分（左连接：包含所有左边表的记录，右连接：包含所有右边表的记录）
            left join tablename on table1.column=table2.column (或选出所有满足条件的table1的列值）
子查询：
  关键词包括： in ,not in , = , !=, exists, not exists
select *from table1 where column1 in (select column1 from table2)
select* from table1 where column1= (select column1 from table2)#此时子查询记录唯一
记录联合 select column1 from table1 union [all] select column1 from table2
UNION 是将结果去重后显示
union all 是将结果不去重直接显示
3. DCL命令
grant select,insert on sakila.*to 'z1‘@’localhost' idenfied by '123'
revoke insert on sakila.* from 'z1'@'localhost'
4. 帮助文档
? 需要帮助的东西即可
5. 查询元数据信息
MySQL5.0后，出现新数据库information_schema，用来记录MySQL的元数据信息。
元数据：数据的数据，比如表名，列名，索引名(statistics)等，实际上物理并不存在， show database 或产生该结果（显示元数据）
