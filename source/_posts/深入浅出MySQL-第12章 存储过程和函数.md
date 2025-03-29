---
title: 深入浅出MySQL-第12章 存储过程和函数
date: 2019-3-1 00:00:00
tags:
- 读书
categories:
- 数据库
---

12.1 定义

1. 存储过程（procedure）和函数（function）是事先经过编译并且存储在数据库的一段SQL语句的集合。
调用存储过程可以减少数据库和应用服务器之间的传输，对于提高数据的处理的效率很有好处。
2. 区别：
函数必须有返回值，参数可以使用IN类型
存储过程没有，参数可以使用 IN,OUT,INOUT类型
IN参数表示这个参数只是作为输入传进来的
OUT表明这个参数是作为输出
INOUT表示参数既可以作为输入也可以作为输出
12.2 存储过程和函数的相关操作
1. 存储过程和函数
创建需要权限：CREATE ROUTINE
修改或者删除： ALTER ROUTINE
执行：EXECUTE
12.2.1 创建，修改存储过程或者函数
CREATE ROUTINE sp_name (proc_parameter[...])
[characteristic ..] routine_body
补充： load data infile用法
用于导入数据
<https://www.cnblogs.com/weiyiming007/p/8125432.html>
load data infile语句从一个文本文件中以很高的速度读入一个表中。使用这个命令之前，mysqld进程（服务）必须已经在运行。由于安全原因，当读取位于服务器上的文件时，文件必须处于数据库目录或可被所有人读取。另外，为了对服务器上文件使用load data infile，在服务器主机上必须有file的权限。
补充：FOUND ROWS用法
ysql 的FOUND_ROWS()
例如需要取出一张表的前10行，同时又需要取出符合条件的总数。这在某些翻页操作中很常见
SELECT SQL_CALC_FOUND_ROWS * FROM tbl_name
WHERE id > 100 LIMIT 10;
在上一查询之后，你只需要用FOUND_ROWS()就能获得查询总数，这个数目是抛掉了LIMIT之后的结果数:
SELECT FOUND_ROWS();
其中第一个sql里面的SQL_CALC_FOUND_ROWS不可省略，它表示需要取得结果数，也是后面使用FOUND_ROWS()函数的铺垫。
