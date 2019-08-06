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

## 卸载系统自带的 `Mariadb`

```sh
rpm -qa|grep mariadb        # 查询出已安装的 mariadb
rpm -e --nodeps filename    # 上面列出的所有文件
rm -f /etc/my.cnf           # 删除配置文件
```

## 创建 `mysql` 用户组

```sh
groupadd mysql
useradd -g mysql mysql
```

## 下载安装包

```sh
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.12-linux-glibc2.12-x86_64.tar.xz
```

## 解压文件到目录 `/usr/local`

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

## 配置 `/etc/my.cnf`

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
expire_logs_days    =10  # 按需设置过期时间，表示保留最近10天的日志
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
expire_logs_days    =10  # 按需设置过期时间，表示保留最近10天的日志
###############################################################################
```

## 初始化数据目录

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

# 这种初始化数据库目录的方式过时了
./mysql_install_db --basedir=/usr/local/mysql --datadir=/data/mysqldata/data1 --user=mysql
./mysql_install_db --basedir=/usr/local/mysql --datadir=/data/mysqldata/data2 --user=mysql

# 新的初始化数据库目录方式，会在终端打印一个临时登入密码。
./mysqld --initialize --basedir=/usr/local/mysql --datadir=/data/mysqldata/data1 --user=mysql --console
./mysqld --initialize --basedir=/usr/local/mysql --datadir=/data/mysqldata/data2 --user=mysql --console


```

## 启动实例

```sh
cd /usr/local/mysql/bin
mysqld_multi start 1
mysqld_multi start 2
mysqld_multi report
```

## 主库创建同步账号

```sh
# 用刚才初始化数据库目录生成的临时密码登入
./mysql -S /var/lib/mysql/mysql1.sock -p your-password

# 如果忘记密码，可以在 my.cnf 的 mysqld 块中添加一行 skip-grant-tables = 1
# 可以进行无密码登入，修改成功之后去掉这一行然后重启数据库。

# 修改 root 密码
USE mysql;
UPDATE user SET authentication_string = PASSWORD('new-password'), password_expired = 'N', password_last_changed = NOW() WHERE user = 'root';
FLUSH PRIVILEGES;

# 授权账户使得局域网内的机器可以访问数据库
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'new-password' WITH GRANT OPTION;

# 创建一个同步账户
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl-password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

# 查看状态
SHOW MASTER STATUS;
SHOW BINARY LOGS;

# 如果忘记设置日志过期时间，可以进入数据库进行全局设置，并手动清理过期日志
# 不要在数据库目录进行删除日志，这样会使得数据库日志索引不一致，导致自动清理失效
SET GLOBAL EXPIRE_LOGS_DAYS = 10;
FLUSH LOGS;  # 触发日志清理，一般是在有新的日志生成的时候触发检查一次。
SHOW BINARY LOGS;

# 也可以手动删除某个日志之前的所有日志
PURGE BINARY LOGS TO 'mysql-bin.000080';  # 删除 80 之前的日志
SHOW BINARY LOGS;

```

## 从库配置

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
