---
title: MySQL日志清理
date: 2019-09-10 14:32
author: Leon
tags: [MySQL, 服务器, Linux]
---

# 一. binlog日志
binlog指二进制日志，它记录了数据库上的所有改变，并以二进制的形式保存在磁盘中，它可以用来查看数据库的变更历史、数据库增量备份和恢复、MySQL的复制（主从数据库的复制）。

个人开发用的服务器以及未开启主从复制的服务器，可以精简binlog从而节省大量磁盘空间。

## 1. 修改MySQL配置文件my.cnf
修改MySQL配置文件my.cnf，配置binlog的过期时间为3天，重启MySQL后生效
```
expire_logs_days=3
```

## 2. 手动删除

手动删除前需要先确认主从库当前在用的binlog文件
```
mysql> show master status;

+---------------+-----------+--------------+------------------+-------------------+
| File          | Position  | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+-----------+--------------+------------------+-------------------+
| binlog.000090 | 141530160 |              |                  |                   |
+---------------+-----------+--------------+------------------+-------------------+
1 row in set (0.03 sec)
```
假设当前在用的binlog文件为binlog.000090，现需要删除binlog.000090之前的所有binlog日志文件(不删binlog.000090)：

```
PURGE MASTER LOGS TO 'master-bin.000277';
```

# 二. mysql-slow.log

如果my.cnf文件中log-queries-not-using-indexes = 1,那么慢查询记录的日志中就不完全是慢查询日志，它包含了查询中没有引用索引的语句，久而久之慢查询日志文件会越来越大

## 正确安全清空在线慢查询日志slowlog的流程

```
# 查看慢查询日志状态 
mysql> show variables like '%slow%';
 
# 关闭慢查询日志 
mysql> set global slow_query_log=0; 
 
mysql> show variables like '%slow%';
 
# 指定新的慢查询日志文件路径 
mysql> set global slow_query_log_file='/var/lib/mysql/slow_queries_new.log';
 
# 开启慢查询日志
mysql> set global slow_query_log=1; 
 
mysql> show variables like '%slow%';

# 在新的慢查询日志文件中，检查慢查询语句 
mysql> select sleep(5) as a, 1 as b; 
 
more /var/lib/mysql/slow_queries_new.log
 
# 备份原有慢查询日志到新的目录
mv /var/lib/mysql/old-slow.log /bakup/old-slow.log.bak               #备份慢查询日志
```
最后删除文件慢查询文件
关闭未走索引查询语句记录slow log
```
set global log_queries_not_using_indexes = 'off';
```






