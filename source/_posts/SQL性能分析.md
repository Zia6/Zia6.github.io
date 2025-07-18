---
title: SQL性能分析
date: 2025-02-20 21:12:18
tags:
categories: MySQL
---
# SQL性能分析(MySQL)
## SQL执行频次查询
MySQL客户端连接成功后，通过show[session | global] status命令可以提供服务器状态信息。通过如下指令，可以查看当前数据库的
INSERT、UPDATE、DELETE、SELECT的访问频次:
```SQL
SHOW GLOBAL STATUS LIKE 'COM______';
```
## 慢查询日志
慢查询日志记录了所有执行时间超过指定参数(long_query_time，单位：秒，默认10秒)的所有SQL语句的日志。
我们可以通过查询慢查询日志来查看是哪些操作比较慢，进而进行优化

MySQL的慢查询日志默认没有开启，需要在MySQL的配置文件(/etc/my.cnf)中配置如下信息：
```SQL
#开启MySQL慢日志查询开关
slow_query_log=1
#设置慢日志的时间为2秒，SQL语句执行时间超过2秒，就会视为慢查询，记录慢查询日志
long_query_time=2
```
配置完毕之后，通过以下指令重新启动MySQL服务器进行测试，查看慢日志文件中记录的信息/var/lib/mysql/localhost-slow.log。

## show profiles
showprofiles能够在做SQL优化时帮助我们了解时间都耗费到哪里去了。通过have_profiling参数，能够看到当前MySQL是否支持。
profile操作：
```SQL
SELECT @@have_profiling;
```
默认profiling是关闭的，可以通过set语句在session/global级别开启profiling:
```SQL
SET profiling = l;
```

执行一系列的业务SQL的操作，然后通过如下指令查看指令的执行耗时：
```SQL
#查看每一条SQL的耗时基本情况
show profiles;
#查看指定query_id的SQL语句各个阶段的耗时情况
show profile for query query_id;
#查看指定query_id的SQL语句CPU的使用情况
show profile cpu for query query_id;
```

## explain执行计划
EXPLAIN或者DESC命令获取MySQL如何执行SELECT语句的信息，包括在SELECT语句执行过程中表如何连接和连接的顺序。
语法:
EXPLAIN SELECT 字段列表 FROM 表名 WHERE 条件；

### 各字段的含义
- id  select查询的序列号，表示查询中执行select子句或者是操作表的顺序(id相同，执行顺序从上到下;id不同，值越大，越先执行)。
- select_type 表示SELECT的类型，常见的取值有SIMPLE（简单表，即不使用表连接或者子查询）、PRIMARY（主查询，即外层的查询）、UNION(UNION中的第二个或者后面的查询语句)、SUBQUERY（SELECT/WHERE之后包含了子查询）等
- type 表示连接类型，性能由好到差的连接类型为NULL、system、const、eq_ref、ref、range、index、all。
- possible_key 显示这张表上可能用得到索引
- Key 实际使用的索引
- Key_len 索引使用的字节数
- rows MySQL认为必须要执行查询的行数，不一定是准确的
- filtered 返回行数占必须执行查询行数的百分比，越大越好
