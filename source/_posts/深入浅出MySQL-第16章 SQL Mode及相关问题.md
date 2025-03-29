---
title: 深入浅出MySQL-第16章 SQL Mode及相关问题
date: 2019-4-6 00:00:00
tags:
- 读书
categories:
- 数据库
---

MySQL可以运行在不同的SQL Mode（SQL模式）下面。
SQL Mode定义了MySQL应支持的SQL语法，数据校验等等。
16.1 MySQL SQL Mode简介

1. 查看SQL Mode
@@sql_mode
2. 如果SQL Mode为STRICT_TRANS_TABLES（严格模式），那么当插入的时候插入的字段长度超过规定，会报错Error。
如果不用这个模式，插入的时候即执行insert的时候，会给出警告warning。但是数据库存的是截断之后的字符串。
设置SQL Mode
set session sql_mode = 'STRICT_TRANS_TABLES'
session表示只在本次连接中生效，也可以替换为global选项，global 表示在本次不生效，但是在新连接生效。
16.1 MySQL SQL Mode简介
16.1 校验日期数据的合法性
如果当前SQL Mode为ANSI,插入非法日期会给出警告，然后实际上数据库存进去的是0000-00-00 00:00:00。
如果更改模式为TRADITIONAL，会直接提示日期格式非法error，拒绝插入。
16.2 MOD(x,0)
更改模式为TRADITIONAL，此时仍然是严格模式，那么会直接给出error。
非严格模式会返回NULL。
16.3 反斜杠\变为普通字符
在ANSI模式中添加NO_BACKSLASH_ESCAPES模式后，可以使得当要插入的字符串中含有反斜杠\时，反斜杠可以被看做为普通字符，不需要在反斜杠前面加上\来进行转义了。具体可以看书上的例子。
16.4 ||视为字符串连接操作符
设置SQL Mode为PIPES_AS_CONCAT,可以将||看做是字符串连接符。
16.2 常用的SQL Mode
sql_mode为ANSI时，等同于READ_AS_FLOAT,PIPES_AS_CONCAT,ANSI_QUOQTES,IGNORE_SPACE,ANSI的组合模式，该模式使得语法和行为更符合标准的MySQL。
sql_mode为STRICT_TRANS_TABLESI时，是严格模式，适用于事务表和非事务表，不允许非法日期，不允许超过长度的字符串，会直接给出错误Error。
sql_mode为TRADITIONAL时，等同于 STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,TRADITIONAL,NO_AUTO_CREATE_USER等等模式的集合，是严格模式，插入不正确的值的时候会报错。用在事务表中，只要出现错误会立刻回滚。
16.3 SQL在迁移中如何使用
可以设置SQL Mode为NO_TABLE_OPTIONS模式，这样当输入show create table的时候，就会不显示最后的ENGINE关键字。
