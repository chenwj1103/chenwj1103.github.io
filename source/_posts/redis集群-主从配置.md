---
title: redis集群-主从配置
copyright: true
date: 2018-02-08 00:57:55
tags: redis 主从
categories: redis

---

由于redis支持的redis cluster和redis sentinel 的集群模式为了保证高可用和高负载需要多台服务器.用redis的主从模式,读写分离,是另一种高可用的方案(容灾效果相对不好).

# redis主从模式的特点

- 一个master可以拥有多个slave
- 多个slave链接同一个master，也可以链接其它slave
- 主从复制不会阻塞master,在同步数据时，master可以继续处理client请求.
- slave 配置为slave-read-only on需要升级为主节点或者写入配置文件中, 而不能在默认slave情况下直接设置
- master与slave断开后会检测心跳, 从新建立连接.
- 可以直接copy DUMP文件从新重启master
- 在Master为空以后，slave同步数据会抹掉全部数据.
- 在实际的使用中最好主服务器还是使用50%到65%的内存,剩下的用于bgsave命令的运行和创建记录写命令的缓冲区.
- 从服务器在同步主服务器的数据的时候,会清空从服务器中的数据,并被替换为主服务器发来的数据.



以下为redis的主从配置实战

# 下载并编译redis源码包

````
wget http://download.redis.io/releases/redis-3.2.10.tar.gz

tar -zxvf redis-3.2.10.tar.gz

cd /redis-3.2.10/src/

make

````

# 配置redis主从

主：127.0.0.1:6379

从：127.0.0.1:6380

````
sudo mkdir redis-6379 redis-6380

sudo cp redis-3.2.10/* -r redis-6379/

sudo cp redis-3.2.10/* -r redis-6380/

sudo mkdir /usr/local/redisSlave/redis-6379/run  /usr/local/redisSlave/redis-6380/run

````
## 配置主节点 redis.config (注意,prod环境需要配置bind 参数)

````
port 6379
daemonize yes
pidfile /usr/local/redisSlave/redis-6379/run/redis_6379.pid

````
## 配置从节点 redis.config

````
port 6380
daemonize yes
pidfile /usr/local/redisSlave/redis-6380/run/redis_6380.pid
slaveof 127.0.0.1 6379

````
其它配置采用默认配置即可

# 启动服务器

````
zhuningning@ubuntu:/usr/local/redisSlave/redis-6379$ sudo ./src/redis-server redis.conf &

zhuningning@ubuntu:/usr/local/redisSlave/redis-6380$ sudo ./src/redis-server redis.conf &

````

# 连接客户端测试

````
zhuningning@ubuntu:/usr/local/redisSlave/redis-6380$ sudo ./src/redis-cli -h 127.0.0.1 -p 6380
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=533,lag=0
master_repl_offset:533
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:532
127.0.0.1:6379> set master 127.0.0.1:6379
OK
127.0.0.1:6379> set slave 127.0.0.1:6380
OK

zhuningning@ubuntu:/usr/local/redisSlave/redis-6379$ sudo ./src/redis-cli -h 127.0.0.1 -p 6379
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:9
master_sync_in_progress:0
slave_repl_offset:750
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
127.0.0.1:6380> get master
"127.0.0.1:6379"
127.0.0.1:6380> get slave
"127.0.0.1:6380"
127.0.0.1:6380> set slave test
(error) READONLY You can't write against a read only slave.
127.0.0.1:6380> 


````
以上信息中role分别代表master 和slave.

在主节点中set master和slave,在从节点中可以获取到值的信息.表示主从配置成功.

# 从节点复制从节点

````
6381配置和6380的一样都是作为6379的丛节点.

6382的配置和6380不同的是,slaveof参数为6381

127.0.0.1:6381> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:8
master_sync_in_progress:0
slave_repl_offset:3284
slave_priority:100
slave_read_only:1
connected_slaves:1
slave0:ip=127.0.0.1,port=6382,state=online,offset=113,lag=0
master_repl_offset:113
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:112


127.0.0.1:6382> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6381
master_link_status:up
master_last_io_seconds_ago:8
master_sync_in_progress:0
slave_repl_offset:99
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0


````

# 主从切换

````
主挂掉了后从执行：
src/redis-cli -p 6380 slaveof NO ONE

主恢复后从执行：
src/redis-cli -p 6380 slaveof 127.0.0.1 6379
````

# 主从配置密码

主:redis.conf中

````
requirepass abc
````

从:redis.conf中

````
masterauth abc
````

