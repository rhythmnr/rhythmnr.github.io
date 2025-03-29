---
title: 深入浅出MySQL-第3章 MySQL支持的数据类型
date: 2019-1-21 00:00:00
tags:
- 读书
categories:
- 数据库
---

1. 数值类型
整数： tinyint,smallint,mediumint, int ,bigint (medium 居然是介质的意思），int默认为int(11)，不满11位用zerofill填充，多了也不截断，传什么就存什么，不会有精度影响。
zerofill 反映在sql语句里面比如：   alter table table1 modify column1 int zerofill
指定为zerofill的列自动加上UNSIGNED属性
AUTO_INCREMENT属性，自增，一般从1开始，一个表最多有一个AUTO_INCREMENT,定义时对应的列为NOT NULL，      primary key 或者 UNIQUE。
小数：浮点数和定点数(decimal)
小数可以用(M,D）表示，M表示整数位，D表示小数位，比如 column float(5,2)，这里如果标度超过D的限制会截断。
BIT（位类型），BIT(M)表示存放多少位二进制，select 的时候写成 bin() （二进制显示）或者 hex() （十六进制显示），不然看不出来
2. 日期时间类型
年月日 DATE
时分秒 TIME
年月日时分秒 DATETIME （默认1970年开始），是DATE和TIME的组合
可以用now（）直接赋值给以上三种类型
MySQL只给第一个TIMESTAMP字段设置默认值为系统日期，第二个TIMESTAMP类型为默认就是0000-00-00了
插入日期的时候会先转换为本地的时区，取出的时候也会转换为本地时区再显示
例子： column1 timestamp not null default current_timestamp
修改当前系统时区： set time_zone = '+9:00' ，会影响timestamp类型的字段
datatime范围比timestamp大
YEAR类型表示年份
3. 字符串类型
检索时候，char删除了尾部的空格，而varchar保留了这些空格。
binary和varbinary与上面两个类型类似，但是它们只包含二进制字符串
枚举值enum忽略大小写，存的时候都是大写，默认存第一个
SET与enum类似，但是set一次性可以选取多个成员，enum只能选一个。
create table table1(column1 set ('a','b','c'))
insert into table1 values ('a','b')
不在范围内会报错，重复存如values ('a','b','c')存进去会过滤掉重复值
