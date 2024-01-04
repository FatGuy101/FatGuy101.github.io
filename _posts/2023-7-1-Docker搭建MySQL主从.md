---
layout: post
title: Docker搭建MySQL主从
subtitle: 一主两从
categories: Linux
author: FatGuy010
permalink: /2023/07/1/DockerBuildsMySQLMaster&Slave
tags: [Docker]
---

# Docker搭建MySQL主从(一主两从)

### 宿主机配置

~~~shell
mkdir /mysql/master/data -p
mkdir /mysql/master/conf -p
mkdir /mysql/slave1/conf -p
mkdir /mysql/slave1/data -p
mkdir /mysql/slave2/data -p
mkdir /mysql/slave2/conf -p
~~~

~~~shell
[root@server153 ~]# tree /mysql/
/mysql/
├── master
│   ├── conf
│   └── data
├── slave1
│   ├── conf
│   └── data
└── slave2
    ├── conf
    └── data

~~~

#### master配置文件

~~~shell
vim /mysql/master/conf/my.cnf

[mysqld]
server_id = 1
log-bin= mysql-bin
read-only=0
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
~~~

#### salve1配置文件

~~~shell
vim /mysql/slave1/conf/my.cnf

[mysqld]
server_id = 2
log-bin= mysql-bin
read-only=1
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
~~~

#### slave2配置文件

~~~shell
vim /mysql/slave2/conf/my.cnf

[mysqld]
server_id = 3
log-bin= mysql-bin
read-only=1
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
~~~

### 启动docker容器

#### 创建Docker桥接网络
~~~shell
docker network create -d bridge mysql
~~~

#### 启动MySQL容器master

~~~shell
docker run --name mysql-master --network mysql -d -e MYSQL_ROOT_PASSWORD=123456 -v /mysql/master/data:/var/lib/mysql -v /mysql/master/conf/my.cnf:/etc/mysql/my.cnf mysql:5.7
~~~

#### 启动MySQL容器slave1

~~~shell
docker run --name mysql-slave1 --network mysql -d -e MYSQL_ROOT_PASSWORD=123456 -v /mysql/slave1/data:/var/lib/mysql -v /mysql/slave1/conf/my.cnf:/etc/mysql/my.cnf mysql:5.7
~~~

#### 启动MySQL容器slave2

~~~shell
docker run --name mysql-slave2 --network mysql -d -e MYSQL_ROOT_PASSWORD=123456 -v /mysql/slave2/data:/var/lib/mysql -v /mysql/slave2/conf/my.cnf:/etc/mysql/my.cnf mysql:5.7
~~~

### MySQL-master

进入master容器

~~~shell
docker exec -it mysql-master /bin/bash
~~~

登录master数据库

~~~shell
mysql -uroot -p123456
~~~

授权账号

~~~mysql
GRANT REPLICATION SLAVE ON *.* to 'slave'@'%' identified by '123456';
~~~

刷新权限

~~~mysql
flush privileges;
~~~

查看master信息

~~~mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      590 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
~~~

### MySQL-slave1

进入slave1容器

~~~shell
docker exec -it mysql-slave1 /bin/bash
~~~

登录slave1数据库

~~~shell
mysql -uroot -p123456
~~~

主从设置

~~~mysql
change master to master_host='mysql-master',master_user='slave',master_password='123456',master_log_file='mysql-bin.000003',master_log_pos=590,master_port=3306;
~~~

~~~mysql
start slave;
~~~

查看MySQL主从配置是否成功

~~~mysql
show slave status\G
~~~

### MySQL-slave2

进入slave2容器

~~~shell
docker exec -it mysql-slave2 /bin/bash
~~~

登录slave1数据库

~~~shell
mysql -uroot -p123456
~~~

主从设置

~~~mysql
change master to master_host='mysql-master',master_user='slave',master_password='123456',master_log_file='mysql-bin.000003',master_log_pos=590,master_port=3306;
~~~

~~~mysql
start slave;
~~~

查看MySQL主从配置是否成功

~~~mysql
show slave status\G
~~~

最后在主库上创库创表，用以验证主从是否搭建成功

### 快速建库建表

~~~MySQL
CREATE DATABASE TESTDB;
show databases;
use TESTDB;
CREATE TABLE TABLE1 (bTypeId int,bName char(16),price int,publishing char(16));
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('1','Linux','66','DZ');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('2','CLD','68','RM');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('3','SYS','90','JX');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('4','MySQL1','71','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('5','MySQL2','72','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('6','MySQL3','73','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('7','MySQL4','74','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('8','MySQL5','75','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('9','MySQL6','76','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('10','MySQL7','77','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('11','MySQL8','78','QH');
INSERT INTO TABLE1(bTypeId,bName,price,publishing) VALUES('12','MySQL9','79','QH');

CREATE TABLE TABLE2 (name char(16),price int,pages int);
INSERT INTO TABLE2(name,price,pages) VALUES('Linux','30','666');
INSERT INTO TABLE2(name,price,pages) VALUES('CLD','60','666');
INSERT INTO TABLE2(name,price,pages) VALUES('SYS','80','666');
INSERT INTO TABLE2(name,price,pages) VALUES('MySQL1','166','666');

show tables;
select * from TABLE1;
select * from TABLE2;
~~~

查看表

~~~mysql
show databases;
use TESTDB;
show tables;
select * from TABLE1;
select * from TABLE2;
~~~

