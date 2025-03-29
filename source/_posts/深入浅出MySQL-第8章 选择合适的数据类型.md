---
title: 深入浅出MySQL-第8章 选择合适的数据类型
date: 2019-1-25 00:00:00
tags:
- 读书
categories:
- 数据库
---

8.1 char与varchar

1. char是固定字节大小的，严格模式下：
如果长度超过这个大小，存进去的时候回直接截断，超过varchar的指定最大大小也直接截断。
char类型读取的时候会去掉末尾的空格，char适用于长度变化不大且对读取速度要求高的数据
InnoDB 建议使用varchar，因为平均上，char比varchar代价大，且InnoDB实际上使用的是指针指向数据值的头。
8.2 text与blob
1. 相对于char与varchar，text和blob保存较大文本。
2. blod能保存二进制数据，如照片等等。
删除操作会在表里面留下“空洞”，这些“空洞”的记录在后续插入性能上会有影响，为了提高性能，应该使用optimize table功能对表进行碎片整理（输入命令 optimize table table1)
补充： MySQL碎片（Fragmentation）
<https://www.databasejournal.com/features/mysql/article.php/3927871/MySQL-Data-Fragmentation---What-When-and-How.htm>
3. 加快文本的查询速度的方法：使用散列值，可以用hash计算后将散列值存在单独的列中。这种方法只适用于精确查找。
4. 如果文本太大，不妨将他们放到单独的表
8.3 浮点数(float)和定点数(decimal)
1. 浮点数插入的精度超过定义的精度，会四舍五入到实际定义的精度，然后插入。
2. 定点数实际上是以字符串形式存储的，精度超过规定后会按照实际精度四舍五入。
3. decimal与浮点数相似，都是(M,N)形式
4. 精度要求高的时候使用定点数
5. 编程中尽量避免浮点数的比较，多用范围比较为而不是==
8.4 日期类型
TIMSTAMP范围没有DATETIME大，但是保存了时区信息。
