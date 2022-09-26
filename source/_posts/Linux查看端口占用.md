---
title: Linux查看端口占用
date: 2018-03-10 17:52
author: Leon
tags: Linux
---
## 1. lsof命令
lsof(list open files)是一个列出当前系统打开文件的工具。

CentOS系统安装lsof:
```
yum install lsof
```

提示“Is this ok[y/d/N]”时，输入y回车

lsof 查看端口占用情况:
```
lsof -i:端口号
```
注意：lsof -i 需要 root 用户的权限来执行

示例：查看3306 端口占用情况
```
lsof -i:3306
```
输出结果

> COMMAND PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
> mysqld  964 mysql   25u  IPv6  17625      0t0  TCP *:mysql (LISTEN)

如上，可以看到MySql服务的相关信息

## 2. netstat命令
netstat查看端口占用情况
```
netstat -tunlp
```
查看3306端口占用情况
```
netstat -tunlp | grep 3306
tcp6       0      0 :::3306            :::*           LISTEN      964/mysqld
```
如上，可以看到mysqld进程以及PID

### netstat命令参数说明

> * -t (tcp) 仅显示tcp相关选项
> * -u (udp)仅显示udp相关选项
> * -n 拒绝显示别名，能显示数字的全部转化为数字
> * -l 仅列出在Listen(监听)的服务状态
> * -p 显示建立相关链接的程序名