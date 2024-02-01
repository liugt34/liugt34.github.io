---
title: PostgreSQL主从复制
date: 2023-06-29 19:57:00 +0800
categories: [PostgreSQL]
tags: [postgresql]
mermaid: true
---

PostgreSQL在9.x之后引入了主从的流复制机制，所谓流复制，就是备服务器通过 **tcp** 流从主服务器中同步相应的数据，主服务器在 **WAL** 记录产生时即将它们以流式传送给备服务器，而不必等到 **WAL** 文件被填充。

1. 默认情况下流复制是异步的，这种情况下主服务器上提交一个事务与该变化在备服务器上变得可见之间客观上存在短暂的延迟，但这种延迟相比基于文件的日志传送方式依然要小得多，在备服务器的能力满足负载的前提下延迟通常低于一秒；
2. 在流复制中，备服务器比使用基于文件的日志传送具有更小的数据丢失窗口，不需要采用archive_timeout来缩减数据丢失窗口；
3. 将一个备服务器从基于文件日志传送转变成基于流复制的步骤是：把recovery.conf文件中的primary_conninfo设置指向主服务器；设置主服务器配置文件的listen_addresses参数与认证文件即可。

# 安装数据库

参考官网 [PostgreSQL: Linux downloads (Ubuntu)](https://www.postgresql.org/download/linux/ubuntu/) 链接

准备两个主机

1. **Master**   IP地址  192.168.1.106
2. **Slave**       IP地址  192.168.1.116

```bash
# Create the file repository configuration:
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update the package lists:
sudo apt-get update

# Install the latest version of PostgreSQL.
# If you want a specific version, use 'postgresql-13' or similar instead of 'postgresql':
sudo apt-get -y install postgresql-13
```



# 在Master节点创建同步账号

```bash
$ psql 
CREATE ROLE replica login replication encrypted password 'replica';
```



# 为账号添加授权

最后一行，添加了replica用户可以从备库IP 192.168.1.116访问主库。

```bash
sudo vim /var/lib/pgsql/13/data/pg_hba.conf

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    all             all             0.0.0.0/0               md5
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
host    replication     replica         192.168.1.116/32        md5
```



# 配置 wal_level

```bash
sudo vim /var/lib/pgsql/13/data/postgresql.conf

# - Settings -
wal_level = logical                    # minimal, replica, or logical
#最多有2个流复制连接
max_wal_senders = 2   
wal_keep_segments = 16  
#流复制超时时间
wal_sender_timeout = 60s
# 最大连接数，据说从机需要大于或等于该值
max_connections = 100 
```

重启数据库

```bash
sudo service postgresql-13 restart
```



# 配置备库

首先在备库上执行对于主库的基础备份

```bash
cd /var/lib/pgsql/13/data
pg_basebackup -h 192.168.1.106 -p 5432 -U replica --password -X stream -Fp --progress -D $PGDATA -R
chmod -R 0700 /var/lib/pgsql/13/data
```

检查备库服务器上是否生成了 **standby.signal** 文件,如果生成了表示备份成功。编辑该文件

```
#开启待机模式
standby_mode = 'on'
```



启动备库

```bash
 pg_ctl start
```

在主库上查询备库信息

```bash
psql -xc "select * from pg_stat_replication"

-[ RECORD 1 ]----+------------------------------
pid              | 21487
usesysid         | 16404
usename          | replica
application_name | walreceiver
client_addr      | 192.168.1.116
client_hostname  | 
client_port      | 43648
backend_start    | 2022-01-10 21:18:57.112831+08
backend_xmin     | 
state            | streaming
sent_lsn         | 0/4000148
write_lsn        | 0/4000148
flush_lsn        | 0/4000148
replay_lsn       | 0/4000148
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2022-01-10 21:23:47.870841+08
```



# 主从切换

主数据库是读写的，备数据库是只读的。当主数据库宕机了，可以通过 **pg_controldata** 命令将从库提升为主库，实现一些基本的HA应用。

查看主库状态

```bash
$ ./pg_controldata /var/lib/pgsql/13/data

pg_control version number:            1300
Catalog version number:               202007201
Database system identifier:           6991633141219620964
Database cluster state:               in production
pg_control last modified:             Thu 01 Feb 2024 09:40:55 AM CST
Latest checkpoint location:           4/DDF69F8
Latest checkpoint's REDO location:    4/DDF69C0
Latest checkpoint's REDO WAL file:    000000010
```

查看从库状态

```bash
$ ./pg_controldata /var/lib/pgsql/13/data

pg_control version number:            1300
Catalog version number:               202007201
Database system identifier:           6991633141219620964
Database cluster state:               archive recovery
```



现在我们将主库停掉并提升从库为主库

```bash
su - postgres -c "pg_ctl promote"
server promoting
```



