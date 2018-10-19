---
title: CentOS 离线安装 MySQL
date: 2018-09-06 11:47:57
categories:
- 笔记
tags:
- MySQL
- CentOS
---

记录一下 CentOS 离线安装 MySQL 并配置多实列主从复制的过程，
如果有旧版 Mariadb，先卸载旧版 Mariadb。

<!-- more -->

<!-- toc -->

# 卸载系统自带的 Mariadb

```sh
rpm -qa|grep mariadb        # 查询出已安装的 mariadb
rpm -e --nodeps filename    # 上面列出的所有文件
rm -f /etc/my.cnf           # 删除配置文件
```

# 创建 mysql 用户组

```sh
groupadd mysql
useradd -g mysql mysql
```

# 下载安装包

```sh
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.12-linux-glibc2.12-x86_64.tar.xz
```

# 解压文件到目录 /usr/local

```sh
cp mysql-8.0.12-linux-glibc2.12-x86_64.tar.xz /usr/local/mysql-8.0.12-linux-glibc2.12-x86_64.tar.xz
cd /usr/local

# xz 结尾的是经过两层压缩的压缩包
# 先解压 xz
xz -d your_file_name.tar.xz

# 再解压 tar
tar -xvf your_file_name.tar

# 或者直接解压
tar xvJf your_file_name.tar.xz

tar xvJf mysql-8.0.12-linux-glibc2.12-x86_64.tar.xz
mv mysql-8.0.12-linux-glibc2.12-x86_64 mysql
```

# 配置 /etc/my.cnf

```sh
[mysqld_multi]
mysqld     = /usr/local/mysql/bin/mysqld_safe
mysqladmin = /usr/local/mysql/bin/mysqladmin
user       = root

# The MySQL server
###############################################################################
[mysqld1]
port                =33306
datadir             =/data/mysqldata/data1
socket              =/var/lib/mysql/mysql1.sock
pid-file            =/var/lib/mysql/mysql1.pid
user                =mysql
server_id           =33306
log_bin             =/data/mysqldata/data1/mysql-bin
log_bin_index       =/data/mysqldata/data1/mysql-bin.index
###############################################################################
[mysqld2]
port                =33307
datadir             =/data/mysqldata/data2
socket              =/var/lib/mysql/mysql2.sock
pid-file            =/var/lib/mysql/mysql2.pid
user                =mysql
server_id           =33307
log_bin             =/data/mysqldata/data2/mysql-bin
log_bin_index       =/data/mysqldata/data2/mysql-bin.index
###############################################################################
```

# 初始化数据目录

```sh
mkdir /var/lib/mysql
chown -R mysql:mysql /var/lib/mysql

mkdir /data/mysqldata
chown -R mysql:mysql /data/mysqldata

mkdir /data/mysqldata/data1
chown -R mysql:mysql /data/mysqldata/data1

mkdir /data/mysqldata/data2
chown -R mysql:mysql /data/mysqldata/data2

cd /usr/local/mysql/bin
./mysql_install_db --basedir=/usr/local/mysql --datadir=/data/mysqldata/data1 --user=mysql
./mysql_install_db --basedir=/usr/local/mysql --datadir=/data/mysqldata/data2 --user=mysql
```

# 启动实例

```sh
cd /usr/local/mysql/bin
mysqld_multi start 1
mysqld_multi start 2
mysqld_multi report
```

# 主库创建同步账号

```sh
./mysql -S /var/lib/mysql/mysql1.sock -p your-password
CREATE USER 'repl'@'%' IDENTIFIED BY 'mysql';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
SHOW MASTER STATUS;
SHOW BINARY LOGS;
```

# 从库配置

```sh
# 修改从库的配置文件
server-id          =2
relay-log          =/dbdata/data/relay-log
relay-log-index    =/dbdata/data/relay-log.index

# 进入数据库执行
CHANGE MASTER TO
MASTER_HOST=‘master_host_name’, # 主库的主机名
MASTER_PORT=port_number # 主库的端口号
MASTER_USER=‘replication_user_name’, # 复制的数据库用户名
MASTER_PASSWORD=‘replication_password’, # 复制的用户密码
MASTER_LOG_FILE=‘recorded_log_file_name’, # 主库的日志文件名
MASTER_LOG_POS=recorded_log_position; # 主库的日志文件位置

start slave;

```
