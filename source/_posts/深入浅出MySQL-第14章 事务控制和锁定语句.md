---
title: 深入浅出MySQL-第14章 事务控制和锁定语句
date: 2019-4-3 00:00:00
tags:
- 读书
categories:
- 数据库
---

MySQL默认对InnoDB进行行级锁定，对MyISAM和MEMORY进行表级锁定。
有时候用户需要明确的进行锁表或者事务的控制，来保证事务的完整性，需要事务控制语句和锁定语句。
14.1 LOCK TABLE和UNLOCK TABLE

1. LOCK TABLE 可以锁定当前线程的表。
语法：
LOCK  TABLES
tbl_name [AS alias] {READ [LOCAL] \ [LOW_PRIORITY] WRITE}
[,tbl_name [AS alias] {READ [LOCAL] \ [LOW_PRIORITY] WRITE}]
(这里的tbl_name 指的是表名）
2. UNLOCK  TABLE 可以释放当前线程的任何锁定。
UNLOCK TABLES （线程释放锁）
如果当前表被一个线程锁定，那么其他线程会一直等待，直到该表被释放。
书上有个非常生动形象的表
14.2 事务控制
1. 默认自动提交(Autocommit)
2. 常用表达
START TRANSACTION | BEGIN [WORK]  
COMMIT [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
ROLLBACK [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
SET AUTOCOMMIT = {0,1}
其中： CHAIN和RELEASE 定义事务提交或者回滚结束之后的操作。CHAIN会立即启动一个事务，并且和刚刚的事务具有相同的隔离级别。RELEASE 会断开和客户端的连接。
3. 如果进行数据库的操作，比如插入数据的时候，写了start transaction了，那么就必须等写commit才提交事务，这个事务结束。这个时候就是不自动提交的，但是如果不写start transaction的话，就表示自动提交，在插入数据结束之后就会自动提交了。
也就是说：start transaction会开始一个新事务，造成一个隐含的unlock tables被执行，也就是说这个时候表被释放了，没有被锁了。
4. 关于日志记录
通常情况下，只对提交的事务记录到二进制日志里面，但是如果一个事务包含非事务类型的表，那么回滚操作也会被记录到二进制日志中，以确保非事务类型表的更新可以被复制到从(Slave）数据库中。
所以的DDL都不能回滚。
5. savepoint
可以定义SAVEPOINT,指定回滚事务的一部分，但是不能指定提交事务的一部分。
对于不需要的SAVEPOINT，可以使用 RELEASE SAVEPOINT 来删除 SAVEPOINT，最后删除的SAVEPOINT不能再执行ROLLBAK TO SAVEPOINT.
命令执行：
savepoint savepointname  # 定义savepoint
rollback savepoint savepointname  #回滚到刚才定义的savepoing，也就是说一直往上，到savepoint savepointname的这一段都被撤销了。
14.3 分布式事务的使用
只有InnoDB支持分布式事务，一个分布式事务设计多个行动。每个行动是一个单独的事务。这些个事务要么一起成功，要么一起失败。
14.3.1 分布式事务的原理
1. 分布式事务需要一个或者多个资源管理器（RM）或者事务管理器（TM）
数据库服务器是一种资源管理器
2. 分布式事务的提交需要两个阶段
第一阶段：所有的分支事务都准备好，也就是说这些事务被TM告知要提交。用于管理分支事务的RM会记录被稳定保存的分支的行动。分支事务指示是否他们可以这么做，这些结果会被用于第二阶段。
第二阶段：TM告知RMs是否要提交或者回滚。如果预备阶段，所有的分支指示他们能够提交，则所有的分支都被告知要提交。如预备阶段有一个分支说不能提交，则TM告知所有分支要回滚。
14.3.2 分布式事务的语法
1,。两个阶段的语法
针对14.3.1.2 的两个阶段，只要使用下面的语法：
第一阶段：
 XA {ATART|BEGIN} xid {JOIN|RESUME}
启动一个带xid值的XA事务。每个事务都有一个唯一的xid值，是XA事务表示符。由客户端生成，或者MySQL服务器生成。
xid 包含以下部分：xid：gtrid[.bequal[,firmatID]]
grid是事务表示符，相同的分布式事务应该使用相同的事务表示符。
bequal 分支限定符，默认为空串，相同的分布式事务中各个分支事务的bequal不应该有一样的。
formatID 表示grid和bequal值的格式，默认为1.
XA END xid [SUSPEND [FOR MIGRATE]]
XA PREPARE xid 事务进入prepare状态
XA RECOVER 可以查看当前状态
第二阶段：
XA COMMIT xid [ONE PHASE] 或者XA ROLLBACK xid
14.3.3 存在的问题
1. 主要的一个例子：
如果一个分支事务已经进入了prepare阶段，这个时候数据库异常启动，启动后虽然可以通过 xa recover看到当前有prepare状态的事务。但是因为该事务还没有提交，写入binlog（二进制日志），所以重启后无法根据binlog恢复之前被修改的数据。对于其他分支事务来说，可能已经提交成功，这将导致分布式事务的不完整性，会丢失部分内容。
