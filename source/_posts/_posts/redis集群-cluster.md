---
title: redis集群-cluster
copyright: true
date: 2018-02-05 00:20:33
tags: redis cluster
categories: redis

---

最近学习了redis设计与实现这本书.这本书中主要讲述是理论知识,所以最近首先整理一下关于redis的集群的知识.然后学习redis实战这本书,实战一下.

# 关于redis cluster

redis cluster在设计的时候，就考虑到了去中心化，去中间件，也就是说，集群中的每个节点都是平等的关系，都是对等的，每个节点都保存各自的数据和整个集群的状态。每个节点都和其他所有节点连接，而且这些连接保持活跃，这样就保证了我们只需要连接集群中的任意一个节点，就可以获取到其他节点的数据。

Redis 集群**没有使用传统的一致性哈希**来分配数据，而是采用另外一种叫做**哈希槽** (hash slot)的方式来分配的。redis cluster 默认分配了 16384 个slot，当我们set一个key 时，会用CRC16算法来取模得到所属的slot，然后将这个key 分到哈希槽区间的节点上，具体算法就是：CRC16(key) % 16384。

# Redis Cluster主从模式

redis cluster 为了保证数据的高可用性，加入了主从模式，一个主节点对应一个或多个从节点，主节点提供数据存取，从节点则是从主节点拉取数据备份，当这个主节点挂掉后，就会有这个从节点选取一个来充当主节点，从而保证集群不会挂掉。

- 例子

集群有ABC三个主节点, 如果这3个节点都没有加入从节点，如果B挂掉了，我们就无法访问整个集群了。A和C的slot也无法访问。所以我们在集群建立的时候，一定要为每个主节点都添加了从节点,比如像这样, 集群包含主节点A、B、C, 以及从节点A1、B1、C1, 那么即使B挂掉系统也可以继续正确工作。B1节点替代了B节点，所以Redis集群将会选择B1节点作为新的主节点，集群将会继续正确地提供服务。当B重新开启后，它就会变成B1的从节点。不过需要注意，如果节点B和B1同时挂了，Redis集群就无法继续正确地提供服务了。

更多关于redisCluster的知识访问我的博文 [redisCluster](http://chenwj.cn/2018-01-28/redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0-cluster/)

# 实战

按照[官方cluster文档](http://redis.io/topics/cluster-tutorial)进行实战 

## 创建并使用集群

Redis 集群由多个运行在集群模式（cluster mode）下的 Redis 实例组成， 实例的集群模式需要通过配置来开启， 开启集群模式的实例将可以使用集群特有的功能和命令。

以下是一个包含了最少选项的集群配置文件示例：

````
port 7000    //端口号
cluster-enabled yes   //开启集群模式
cluster-config-file nodes.conf   //集群的配置文件
cluster-node-timeout 5000   //认定节点下线的时间
appendonly yes   //开启aof持久化

````

官方建议三主三从来搭建集群(本集群采用3.2.11版本)

首先下载redis的源码 地址[redis-3.2.11](http://download.redis.io/releases/redis-3.2.11.tar.gz)

- 下载源码 修改文件权限
````
wget http://download.redis.io/releases/redis-3.2.11.tar.gz
sudo chmod  753 redis-3.2.11.tar.gz 
````

创建集群文件夹  解压redis源码 进入redis集群文件夹下的redis-3.2.11
````
sudo mkdir /usr/local/redisCluster
sudo tar -zxvf redis-3.2.11.tar.gz -C /usr/local/redisCluster/
cd /usr/local/redisCluster/redis-3.2.11/
````

- 安装(该操作需要linux系统有c环境,如果报错请自行查阅资料安装) 以上操作后在src文件夹下会生成可执行文件 redis-server和redis-cli等,如果出现编译问题,请切换到root用户下操作
````
sudo make&&make install
````

- 然后修改配置文件
````
sudo vim redis.conf

如下

port 7000    //端口号
cluster-enabled yes   //开启集群模式
cluster-config-file nodes.conf   //集群的配置文件
cluster-node-timeout 5000   //认定节点下线的时间
appendonly yes   //开启aof持久化

````

- 让我们进入一个新目录， 并创建六个以端口号为名字的子目录， 稍后我们在将每个目录中运行一个 Redis 实例

````
cd ..
sudo mkdir 7000 7001 7002 7003 7004 7005 

/usr/local/redisCluster$ sudo cp -r redis-3.2.11/*  7000/
/usr/local/redisCluster$ sudo cp -r redis-3.2.11/*  7001/
/usr/local/redisCluster$ sudo cp -r redis-3.2.11/*  7002/
/usr/local/redisCluster$ sudo cp -r redis-3.2.11/*  7003/
/usr/local/redisCluster$ sudo cp -r redis-3.2.11/*  7004/
/usr/local/redisCluster$ sudo cp -r redis-3.2.11/*  7005/

现在只需修改每个配置文件下的port参数就行    

````

- 然后进入每个端口号命名的文件夹下运行以下命令

````
sudo ./src/redis-server redis.conf &

zhuningning@ubuntu:/usr/local/redisCluster/7005$ ps -ef|grep redis
root      4700  3041  0 00:56 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4701  4700  0 00:56 pts/4    00:00:00 ./src/redis-server 127.0.0.1:7003 [cluster]
root      4721  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4722  4721  0 00:57 pts/4    00:00:00 ./src/redis-server 127.0.0.1:7000 [cluster]
root      4740  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4741  4740  0 00:57 pts/4    00:00:00 ./src/redis-server 127.0.0.1:7001 [cluster]
root      4755  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4756  4755  0 00:57 pts/4    00:00:00 ./src/redis-server 127.0.0.1:7002 [cluster]
root      4771  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4772  4771  0 00:57 pts/4    00:00:00 ./src/redis-server 127.0.0.1:7004 [cluster]
root      4789  3041  0 00:58 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4790  4789  0 00:58 pts/4    00:00:00 ./src/redis-server 127.0.0.1:7005 [cluster]
zhuning+  4794  3041  0 00:58 pts/4    00:00:00 grep --color=auto redis
zhuningning@ubuntu:/usr/local/redisCluster/7005$ 

````
- 使用 Redis 集群命令行工具 redis-trib ， 编写节点配置文件的工作可以非常容易地完成

````

zhuningning@ubuntu:/usr/local/redisCluster/7005$ sudo ./src/redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
M: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
S: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   replicates a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
Can I set the above configuration? (type 'yes' to accept): 


````

其中--replicas 1表示每个主节点创建一个从节点,该命令自动为各个主节点分配槽位.m表示主节点,s表示从节点;

yes确认

````
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
4722:M 06 Feb 01:04:46.722 # configEpoch set to 1 via CLUSTER SET-CONFIG-EPOCH
4741:M 06 Feb 01:04:46.722 # configEpoch set to 2 via CLUSTER SET-CONFIG-EPOCH
4756:M 06 Feb 01:04:46.723 # configEpoch set to 3 via CLUSTER SET-CONFIG-EPOCH
4701:M 06 Feb 01:04:46.723 # configEpoch set to 4 via CLUSTER SET-CONFIG-EPOCH
4772:M 06 Feb 01:04:46.724 # configEpoch set to 5 via CLUSTER SET-CONFIG-EPOCH
4790:M 06 Feb 01:04:46.724 # configEpoch set to 6 via CLUSTER SET-CONFIG-EPOCH
>>> Sending CLUSTER MEET messages to join the cluster
4722:M 06 Feb 01:04:46.747 # IP address for this node updated to 127.0.0.1
4741:M 06 Feb 01:04:46.801 # IP address for this node updated to 127.0.0.1
4790:M 06 Feb 01:04:46.801 # IP address for this node updated to 127.0.0.1
4701:M 06 Feb 01:04:46.801 # IP address for this node updated to 127.0.0.1
4756:M 06 Feb 01:04:46.802 # IP address for this node updated to 127.0.0.1
4772:M 06 Feb 01:04:46.802 # IP address for this node updated to 127.0.0.1
Waiting for the cluster to join...
4701:S 06 Feb 01:04:50.766 # Cluster state changed: ok
4772:S 06 Feb 01:04:50.768 # Cluster state changed: ok
4790:S 06 Feb 01:04:50.769 # Cluster state changed: ok
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
S: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots: (0 slots) slave
   replicates a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
zhuningning@ubuntu:/usr/local/redisCluster/7005$ 4701:S 06 Feb 01:04:51.318 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:04:51.318 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:04:51.318 * Non blocking connect for SYNC fired the event.
4701:S 06 Feb 01:04:51.318 * Master replied to PING, replication can continue...
4701:S 06 Feb 01:04:51.319 * Partial resynchronization not possible (no cached master)
4722:M 06 Feb 01:04:51.319 * Slave 127.0.0.1:7003 asks for synchronization
4722:M 06 Feb 01:04:51.319 * Full resync requested by slave 127.0.0.1:7003
4722:M 06 Feb 01:04:51.319 * Starting BGSAVE for SYNC with target: disk
4722:M 06 Feb 01:04:51.320 * Background saving started by pid 5685
4701:S 06 Feb 01:04:51.321 * Full resync from master: 4e9540a7b3c5c6caeb88fd51d09a1ecfc067fd5b:1
5685:C 06 Feb 01:04:51.348 * DB saved on disk
5685:C 06 Feb 01:04:51.350 * RDB: 6 MB of memory used by copy-on-write
4722:M 06 Feb 01:04:51.426 * Background saving terminated with success
4722:M 06 Feb 01:04:51.426 * Synchronization with slave 127.0.0.1:7003 succeeded
4701:S 06 Feb 01:04:51.426 * MASTER <-> SLAVE sync: receiving 77 bytes from master
4701:S 06 Feb 01:04:51.427 * MASTER <-> SLAVE sync: Flushing old data
4701:S 06 Feb 01:04:51.427 * MASTER <-> SLAVE sync: Loading DB in memory
4701:S 06 Feb 01:04:51.427 * MASTER <-> SLAVE sync: Finished with success
4701:S 06 Feb 01:04:51.428 * Background append only file rewriting started by pid 5686
4701:S 06 Feb 01:04:51.462 * AOF rewrite child asks to stop sending diffs.
5686:C 06 Feb 01:04:51.462 * Parent agreed to stop sending diffs. Finalizing AOF...
5686:C 06 Feb 01:04:51.462 * Concatenating 0.00 MB of AOF diff received from parent.
5686:C 06 Feb 01:04:51.463 * SYNC append only file rewrite performed
5686:C 06 Feb 01:04:51.463 * AOF rewrite: 6 MB of memory used by copy-on-write
4701:S 06 Feb 01:04:51.519 * Background AOF rewrite terminated with success
4701:S 06 Feb 01:04:51.520 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
4701:S 06 Feb 01:04:51.520 * Background AOF rewrite finished successfully
4772:S 06 Feb 01:04:51.527 * Connecting to MASTER 127.0.0.1:7001
4772:S 06 Feb 01:04:51.527 * MASTER <-> SLAVE sync started
4772:S 06 Feb 01:04:51.527 * Non blocking connect for SYNC fired the event.
4772:S 06 Feb 01:04:51.528 * Master replied to PING, replication can continue...
4772:S 06 Feb 01:04:51.529 * Partial resynchronization not possible (no cached master)
4741:M 06 Feb 01:04:51.529 * Slave 127.0.0.1:7004 asks for synchronization
4741:M 06 Feb 01:04:51.529 * Full resync requested by slave 127.0.0.1:7004
4741:M 06 Feb 01:04:51.529 * Starting BGSAVE for SYNC with target: disk
4741:M 06 Feb 01:04:51.532 * Background saving started by pid 5687
4772:S 06 Feb 01:04:51.535 * Full resync from master: be30d158d2de1e2edf37e74dc2c3cee348d02410:1
5687:C 06 Feb 01:04:51.565 * DB saved on disk
5687:C 06 Feb 01:04:51.566 * RDB: 6 MB of memory used by copy-on-write
4790:S 06 Feb 01:04:51.584 * Connecting to MASTER 127.0.0.1:7002
4790:S 06 Feb 01:04:51.584 * MASTER <-> SLAVE sync started
4790:S 06 Feb 01:04:51.584 * Non blocking connect for SYNC fired the event.
4790:S 06 Feb 01:04:51.585 * Master replied to PING, replication can continue...
4790:S 06 Feb 01:04:51.585 * Partial resynchronization not possible (no cached master)
4756:M 06 Feb 01:04:51.585 * Slave 127.0.0.1:7005 asks for synchronization
4756:M 06 Feb 01:04:51.585 * Full resync requested by slave 127.0.0.1:7005
4756:M 06 Feb 01:04:51.585 * Starting BGSAVE for SYNC with target: disk
4756:M 06 Feb 01:04:51.586 * Background saving started by pid 5688
4790:S 06 Feb 01:04:51.587 * Full resync from master: 11082aa273a96447d9bdad8a6a004ea85878ad65:1
4741:M 06 Feb 01:04:51.599 * Background saving terminated with success
4772:S 06 Feb 01:04:51.600 * MASTER <-> SLAVE sync: receiving 77 bytes from master
4741:M 06 Feb 01:04:51.600 * Synchronization with slave 127.0.0.1:7004 succeeded
4772:S 06 Feb 01:04:51.600 * MASTER <-> SLAVE sync: Flushing old data
4772:S 06 Feb 01:04:51.600 * MASTER <-> SLAVE sync: Loading DB in memory
4772:S 06 Feb 01:04:51.600 * MASTER <-> SLAVE sync: Finished with success
5688:C 06 Feb 01:04:51.624 * DB saved on disk
4772:S 06 Feb 01:04:51.625 * Background append only file rewriting started by pid 5689
5688:C 06 Feb 01:04:51.625 * RDB: 6 MB of memory used by copy-on-write
4722:M 06 Feb 01:04:51.627 # Cluster state changed: ok
4772:S 06 Feb 01:04:51.663 * AOF rewrite child asks to stop sending diffs.
5689:C 06 Feb 01:04:51.663 * Parent agreed to stop sending diffs. Finalizing AOF...
5689:C 06 Feb 01:04:51.663 * Concatenating 0.00 MB of AOF diff received from parent.
5689:C 06 Feb 01:04:51.663 * SYNC append only file rewrite performed
5689:C 06 Feb 01:04:51.664 * AOF rewrite: 6 MB of memory used by copy-on-write
4756:M 06 Feb 01:04:51.665 * Background saving terminated with success
4756:M 06 Feb 01:04:51.665 # Cluster state changed: ok
4790:S 06 Feb 01:04:51.665 * MASTER <-> SLAVE sync: receiving 77 bytes from master
4756:M 06 Feb 01:04:51.665 * Synchronization with slave 127.0.0.1:7005 succeeded
4790:S 06 Feb 01:04:51.665 * MASTER <-> SLAVE sync: Flushing old data
4790:S 06 Feb 01:04:51.665 * MASTER <-> SLAVE sync: Loading DB in memory
4790:S 06 Feb 01:04:51.665 * MASTER <-> SLAVE sync: Finished with success
4790:S 06 Feb 01:04:51.666 * Background append only file rewriting started by pid 5690
4741:M 06 Feb 01:04:51.699 # Cluster state changed: ok
4790:S 06 Feb 01:04:51.705 * AOF rewrite child asks to stop sending diffs.
5690:C 06 Feb 01:04:51.705 * Parent agreed to stop sending diffs. Finalizing AOF...
5690:C 06 Feb 01:04:51.706 * Concatenating 0.00 MB of AOF diff received from parent.
5690:C 06 Feb 01:04:51.706 * SYNC append only file rewrite performed
5690:C 06 Feb 01:04:51.707 * AOF rewrite: 4 MB of memory used by copy-on-write
4772:S 06 Feb 01:04:51.727 * Background AOF rewrite terminated with success
4772:S 06 Feb 01:04:51.727 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
4772:S 06 Feb 01:04:51.727 * Background AOF rewrite finished successfully
4790:S 06 Feb 01:04:51.785 * Background AOF rewrite terminated with success
4790:S 06 Feb 01:04:51.785 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
4790:S 06 Feb 01:04:51.785 * Background AOF rewrite finished successfully

````
以上为创建集群成功后的整个打印信息.

以下信息表示集群创建成功.

````
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
````

连接上集群,查看集群信息

````
zhuningning@ubuntu:/usr/local/redisCluster/7005$ ./src/redis-cli -h 127.0.0.1 -p 7005
127.0.0.1:7005> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:3
cluster_stats_messages_sent:1559
cluster_stats_messages_received:1559

````
## 检查集群

````
zhuningning@ubuntu:/usr/local/redisCluster/7005$ ./src/redis-cli -c -p 7005
127.0.0.1:7005> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
127.0.0.1:7002> set test test
-> Redirected to slot [6918] located at 127.0.0.1:7001
OK
127.0.0.1:7001> get test
"test"

````
上面参数 -c表示集群客户端

````
127.0.0.1:7001> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
127.0.0.1:7002> get foo
"bar"

````

注意以上语句 7002节点上存在foo/bar和cluster/test键值对,我们查看它对应的从节点7005的aof中存在的数据.

````
zhuningning@ubuntu:/usr/local/redisCluster/7005$ cat -n appendonly.aof 
     1	*2
     2	$6
     3	SELECT
     4	$1
     5	0
     6	*3
     7	$3
     8	set
     9	$3
    10	foo
    11	$3
    12	bar
    13	*3
    14	$3
    15	set
    16	$7
    17	cluster
    18	$4
    19	test

````
aof中显示,7005节点存在两组键值对,表示主节点的数据同步到从节点上.

## 集群故障

查看集群的情况

````
127.0.0.1:7005> cluster nodes
442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005 myself,slave bb31a89640b381e8de18f15a845512f08ab9f16f 0 0 6 connected
df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003 slave a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 0 1517851933524 4 connected
0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001 master - 0 1517851932019 2 connected 5461-10922
bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002 master - 0 1517851933023 3 connected 10923-16383
37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004 slave 0ca3673dfd352052b651b3898ba3f807ff9c7f55 0 1517851932019 5 connected
a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000 master - 0 1517851933023 1 connected 0-5460
````
此时kill掉7000

````
zhuningning@ubuntu:/usr/local/redisCluster/7005$ ps -ef|grep redis
root      4700  3041  0 00:56 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4701  4700  0 00:56 pts/4    00:00:04 ./src/redis-server 127.0.0.1:7003 [cluster]
root      4721  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4722  4721  0 00:57 pts/4    00:00:04 ./src/redis-server 127.0.0.1:7000 [cluster]
root      4740  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4741  4740  0 00:57 pts/4    00:00:04 ./src/redis-server 127.0.0.1:7001 [cluster]
root      4755  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4756  4755  0 00:57 pts/4    00:00:04 ./src/redis-server 127.0.0.1:7002 [cluster]
root      4771  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4772  4771  0 00:57 pts/4    00:00:04 ./src/redis-server 127.0.0.1:7004 [cluster]
root      4789  3041  0 00:58 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4790  4789  0 00:58 pts/4    00:00:04 ./src/redis-server 127.0.0.1:7005 [cluster]
zhuning+ 10601  3041  0 01:33 pts/4    00:00:00 grep --color=auto redis
zhuningning@ubuntu:/usr/local/redisCluster/7005$ sudo kill -9 4722
[sudo] password for zhuningning: 
4701:S 06 Feb 01:34:11.707 # Connection with master lost.
4701:S 06 Feb 01:34:11.707 * Caching the disconnected master state.
zhuningning@ubuntu:/usr/local/redisCluster/7005$ 4701:S 06 Feb 01:34:12.163 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:34:12.163 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:34:12.163 # Error condition on socket for SYNC: Connection refused
4701:S 06 Feb 01:34:13.168 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:34:13.169 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:34:13.169 # Error condition on socket for SYNC: Connection refused
4701:S 06 Feb 01:34:14.174 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:34:14.175 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:34:14.175 # Error condition on socket for SYNC: Connection refused
4701:S 06 Feb 01:34:15.180 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:34:15.181 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:34:15.181 # Error condition on socket for SYNC: Connection refused
4701:S 06 Feb 01:34:16.186 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:34:16.186 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:34:16.186 # Error condition on socket for SYNC: Connection refused
4701:S 06 Feb 01:34:17.190 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:34:17.190 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:34:17.190 # Error condition on socket for SYNC: Connection refused
4790:S 06 Feb 01:34:17.769 * Marking node a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 as failing (quorum reached).
4790:S 06 Feb 01:34:17.769 # Cluster state changed: fail
4756:M 06 Feb 01:34:18.088 * Marking node a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 as failing (quorum reached).
4756:M 06 Feb 01:34:18.088 # Cluster state changed: fail
4741:M 06 Feb 01:34:18.089 * FAIL message received from bb31a89640b381e8de18f15a845512f08ab9f16f about a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4
4701:S 06 Feb 01:34:18.089 * FAIL message received from bb31a89640b381e8de18f15a845512f08ab9f16f about a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4
4772:S 06 Feb 01:34:18.089 * FAIL message received from bb31a89640b381e8de18f15a845512f08ab9f16f about a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4
4701:S 06 Feb 01:34:18.089 # Cluster state changed: fail
4741:M 06 Feb 01:34:18.089 # Cluster state changed: fail
4772:S 06 Feb 01:34:18.089 # Cluster state changed: fail
4701:S 06 Feb 01:34:18.094 # Start of election delayed for 515 milliseconds (rank #0, offset 2465).
4701:S 06 Feb 01:34:18.194 * Connecting to MASTER 127.0.0.1:7000
4701:S 06 Feb 01:34:18.194 * MASTER <-> SLAVE sync started
4701:S 06 Feb 01:34:18.194 # Error condition on socket for SYNC: Connection refused
4701:S 06 Feb 01:34:18.697 # Starting a failover election for epoch 7.
4756:M 06 Feb 01:34:18.740 # Failover auth granted to df9ac983886f0f09b0c62feef9288a9296a1a08c for epoch 7
4741:M 06 Feb 01:34:18.740 # Failover auth granted to df9ac983886f0f09b0c62feef9288a9296a1a08c for epoch 7
4701:S 06 Feb 01:34:18.740 # Failover election won: I'm the new master.
4701:S 06 Feb 01:34:18.740 # configEpoch set to 7 after successful failover
4701:M 06 Feb 01:34:18.740 * Discarding previously cached master state.
4701:M 06 Feb 01:34:18.740 # Cluster state changed: ok
4756:M 06 Feb 01:34:18.782 # Cluster state changed: ok
4772:S 06 Feb 01:34:18.782 # Cluster state changed: ok
4741:M 06 Feb 01:34:18.782 # Cluster state changed: ok
4790:S 06 Feb 01:34:18.782 # Cluster state changed: ok

````

以上信息表示,在五秒的尝试连接7000节点,发现其连接不上.某个节点首先将其标识为fail状态,然后通过节点间通讯,各个节点都将7000节点对应的状态置为fail状态,然后各个节点将集群状态标识为fail状态.然后各个节点为7000节点的从节点投票选举出7003(在这7000只有一个从节点)为主节点,代替元7000的工作(管理其对应的槽位),然后集群状态恢复.

此时集群状态如下: 7000 节点此时为fail状态,集群中正常的节点有5个.

````
127.0.0.1:7005> cluster nodes
442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005 myself,slave bb31a89640b381e8de18f15a845512f08ab9f16f 0 0 6 connected
df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003 master - 0 1517852576553 7 connected 0-5460
0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001 master - 0 1517852576054 2 connected 5461-10922
bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002 master - 0 1517852575048 3 connected 10923-16383
37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004 slave 0ca3673dfd352052b651b3898ba3f807ff9c7f55 0 1517852575552 5 connected
a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000 master,fail - 1517852051773 1517852049869 1 disconnected

````

## 故障节点恢复


恢复节点7000

````
zhuningning@ubuntu:/usr/local/redisCluster/7000$ sudo ./src/redis-server redis.conf &

13569:M 06 Feb 01:45:53.163 # Server started, Redis version 3.2.11
13569:M 06 Feb 01:45:53.163 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
13569:M 06 Feb 01:45:53.163 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
13569:M 06 Feb 01:45:53.164 * The server is now ready to accept connections on port 7000
13569:M 06 Feb 01:45:53.165 # Configuration change detected. Reconfiguring myself as a replica of df9ac983886f0f09b0c62feef9288a9296a1a08c
13569:S 06 Feb 01:45:53.166 # Cluster state changed: ok
4756:M 06 Feb 01:45:53.247 * Clear FAIL state for node a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4: master without slots is reachable again.
4790:S 06 Feb 01:45:53.247 * Clear FAIL state for node a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4: master without slots is reachable again.
4772:S 06 Feb 01:45:53.254 * Clear FAIL state for node a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4: master without slots is reachable again.
4701:M 06 Feb 01:45:53.259 * Clear FAIL state for node a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4: master without slots is reachable again.
4741:M 06 Feb 01:45:53.261 * Clear FAIL state for node a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4: master without slots is reachable again.
13569:S 06 Feb 01:45:54.168 * Connecting to MASTER 127.0.0.1:7003
13569:S 06 Feb 01:45:54.168 * MASTER <-> SLAVE sync started
13569:S 06 Feb 01:45:54.168 * Non blocking connect for SYNC fired the event.
13569:S 06 Feb 01:45:54.168 * Master replied to PING, replication can continue...
13569:S 06 Feb 01:45:54.169 * Partial resynchronization not possible (no cached master)
4701:M 06 Feb 01:45:54.169 * Slave 127.0.0.1:7000 asks for synchronization
4701:M 06 Feb 01:45:54.169 * Full resync requested by slave 127.0.0.1:7000
4701:M 06 Feb 01:45:54.169 * Starting BGSAVE for SYNC with target: disk
4701:M 06 Feb 01:45:54.170 * Background saving started by pid 13572
13569:S 06 Feb 01:45:54.171 * Full resync from master: cecbe45078fac08152e9ca95d7741ae086b4ea68:1
13572:C 06 Feb 01:45:54.218 * DB saved on disk
13572:C 06 Feb 01:45:54.219 * RDB: 6 MB of memory used by copy-on-write
4701:M 06 Feb 01:45:54.261 * Background saving terminated with success
13569:S 06 Feb 01:45:54.261 * MASTER <-> SLAVE sync: receiving 77 bytes from master
4701:M 06 Feb 01:45:54.261 * Synchronization with slave 127.0.0.1:7000 succeeded
13569:S 06 Feb 01:45:54.262 * MASTER <-> SLAVE sync: Flushing old data
13569:S 06 Feb 01:45:54.262 * MASTER <-> SLAVE sync: Loading DB in memory
13569:S 06 Feb 01:45:54.262 * MASTER <-> SLAVE sync: Finished with success
13569:S 06 Feb 01:45:54.281 * Background append only file rewriting started by pid 13573
13569:S 06 Feb 01:45:54.333 * AOF rewrite child asks to stop sending diffs.
13573:C 06 Feb 01:45:54.333 * Parent agreed to stop sending diffs. Finalizing AOF...
13573:C 06 Feb 01:45:54.333 * Concatenating 0.00 MB of AOF diff received from parent.
13573:C 06 Feb 01:45:54.334 * SYNC append only file rewrite performed
13573:C 06 Feb 01:45:54.334 * AOF rewrite: 4 MB of memory used by copy-on-write
13569:S 06 Feb 01:45:54.382 * Background AOF rewrite terminated with success
13569:S 06 Feb 01:45:54.382 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
13569:S 06 Feb 01:45:54.382 * Background AOF rewrite finished successfully

````

以下两种方式检查集群的状态

````
zhuningning@ubuntu:/usr/local/redisCluster/7000$ ./src/redis-cli -c -p 7000
127.0.0.1:7000> cluster nodes
37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004 slave 0ca3673dfd352052b651b3898ba3f807ff9c7f55 0 1517852872929 5 connected
df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003 master - 0 1517852872427 7 connected 0-5460
442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005 slave bb31a89640b381e8de18f15a845512f08ab9f16f 0 1517852873431 6 connected
bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002 master - 0 1517852873431 3 connected 10923-16383
0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001 master - 0 1517852871424 2 connected 5461-10922
a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000 myself,slave df9ac983886f0f09b0c62feef9288a9296a1a08c 0 0 1 connected
127.0.0.1:7000> quit
zhuningning@ubuntu:/usr/local/redisCluster/7000$ ./src/redis-trib.rb check 127.0.0.1:7001
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots: (0 slots) slave
   replicates df9ac983886f0f09b0c62feef9288a9296a1a08c
M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

````

此时7000节点变为7003节点的从节点

## 集群中新加入删除节点

### 作为主节点加入

新复制一个节点7006,修改其端口号配置,然后启动.

````
zhuningning@ubuntu:/usr/local/redisCluster$ sudo mkdir 7006 
zhuningning@ubuntu:/usr/local/redisCluster$ sudo cp redis-3.2.11/* -r 7006/
zhuningning@ubuntu:/usr/local/redisCluster$ cd 7006
zhuningning@ubuntu:/usr/local/redisCluster/7006$ sudo ./src/redis-server redis.conf &
[8] 15048

zhuningning@ubuntu:/usr/local/redisCluster/7006$ ps -ef|grep redis
root      4700  3041  0 00:56 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4701  4700  0 00:56 pts/4    00:00:09 ./src/redis-server 127.0.0.1:7003 [cluster]
root      4740  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4741  4740  0 00:57 pts/4    00:00:09 ./src/redis-server 127.0.0.1:7001 [cluster]
root      4755  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4756  4755  0 00:57 pts/4    00:00:09 ./src/redis-server 127.0.0.1:7002 [cluster]
root      4771  3041  0 00:57 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4772  4771  0 00:57 pts/4    00:00:08 ./src/redis-server 127.0.0.1:7004 [cluster]
root      4789  3041  0 00:58 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root      4790  4789  0 00:58 pts/4    00:00:09 ./src/redis-server 127.0.0.1:7005 [cluster]
root     13568  3041  0 01:45 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root     13569 13568  0 01:45 pts/4    00:00:01 ./src/redis-server 127.0.0.1:7000 [cluster]
root     15048  3041  0 01:55 pts/4    00:00:00 sudo ./src/redis-server redis.conf
root     15049 15048  0 01:55 pts/4    00:00:00 ./src/redis-server 127.0.0.1:7006 [cluster]
zhuning+ 15139  3041  0 01:55 pts/4    00:00:00 grep --color=auto redis


````

加入新节点7006

````
zhuningning@ubuntu:/usr/local/redisCluster/7006$ ./src/redis-trib.rb add-node 127.0.0.1:7006 127.0.0.1:7000

>>> Adding node 127.0.0.1:7006 to cluster 127.0.0.1:7000
>>> Performing Cluster Check (using node 127.0.0.1:7000)
S: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots: (0 slots) slave
   replicates df9ac983886f0f09b0c62feef9288a9296a1a08c
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 127.0.0.1:7006 to make it join the cluster.
[OK] New node added correctly.
zhuningning@ubuntu:/usr/local/redisCluster/7006$ 15049:M 06 Feb 01:57:40.626 # IP address for this node updated to 127.0.0.1
zhuningning@ubuntu:/usr/local/redisCluster/7006$ 15049:M 06 Feb 01:57:45.733 # Cluster state changed: ok
````

add-node是加入指令，127.0.0.1:7006 表示新加入的节点，127.0.0.1:7000 表示加入的集群的一个节点，用来辨识是哪个集群，理论上哪个都可以。


检查集群状态,此时7006为master节点,处于无槽位状态.

````
zhuningning@ubuntu:/usr/local/redisCluster/7006$ ./src/redis-trib.rb check 127.0.0.1:7006
>>> Performing Cluster Check (using node 127.0.0.1:7006)
M: 54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006
   slots: (0 slots) master
   0 additional replica(s)
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots: (0 slots) slave
   replicates df9ac983886f0f09b0c62feef9288a9296a1a08c
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

````

看来，redis cluster 不是在新加节点的时候帮我们做好了迁移工作，需要我们手动对集群进行重新分片迁移，也是这个命令：

````
zhuningning@ubuntu:/usr/local/redisCluster/7006$ ./src/redis-trib.rb reshard 127.0.0.1:7000
>>> Performing Cluster Check (using node 127.0.0.1:7000)
S: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots: (0 slots) slave
   replicates df9ac983886f0f09b0c62feef9288a9296a1a08c
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: 54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006
   slots: (0 slots) master
   0 additional replica(s)
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 4096
What is the receiving node ID? efc3131fbdc6cf929720e0e0f7136cae85657481
*** The specified node is not known or not a master, please retry.
What is the receiving node ID? 54fe323634d8830f2ef93feaaedb2b70e19d0765
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:all

Ready to move 4096 slots.
  Source nodes:
    M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
    M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
    M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
  Destination node:
    M: 54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006
   slots: (0 slots) master
   0 additional replica(s)
  Resharding plan:
    Moving slot 5461 from 0ca3673dfd352052b651b3898ba3f807ff9c7f55


````

第一个问句 how many slots do you want to move 平均的4096个

第二个问句,接收节点的id :7006对应的节点id:efc3131fbdc6cf929720e0e0f7136cae85657481

第三个问句,从哪获取这些槽位,  如果我们不打算从特定的节点上取出指定数量的哈希槽， 那么可以向 redis-trib 输入 all ， 这样的话， 集群中的所有主节点都会成为源节点.

它会提示是否迁移,输入yes即可,但是最后出现如下的error,查阅资料说时cluster的 [bug](https://github.com/antirez/redis/issues/4272)

````
[ERR] Calling MIGRATE: ERR Syntax error, try CLIENT (LIST | KILL | GETNAME | SETNAME | PAUSE | REPLY)
zhuningning@ubuntu:/usr/local/redisCluster/7006$ ./src/redis-trib.rb reshard 127.0.0.1:7000
>>> Performing Cluster Check (using node 127.0.0.1:7000)
S: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots: (0 slots) slave
   replicates df9ac983886f0f09b0c62feef9288a9296a1a08c
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
M: 54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006
   slots:0-1364,5461-6826,10923-12181 (3990 slots) master
   0 additional replica(s)
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:12182-16383 (4202 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
[WARNING] Node 127.0.0.1:7006 has slots in importing state (12182).
[WARNING] Node 127.0.0.1:7002 has slots in migrating state (12182).
[WARNING] The following slots are open: 12182
>>> Check slots coverage...
[OK] All 16384 slots covered.
*** Please fix your cluster problems before resharding

````



12182这个数字看着比较熟悉,原来是之前的该槽位中有数据.暂时还么有解决.(???????????)

````
zhuningning@ubuntu:/usr/local/redisCluster/7006$ ./src/redis-cli -c -p 7006
127.0.0.1:7006> cluster nodes
37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004 slave 0ca3673dfd352052b651b3898ba3f807ff9c7f55 0 1517854904123 2 connected
df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003 master - 0 1517854903119 7 connected 1365-5460
442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005 slave bb31a89640b381e8de18f15a845512f08ab9f16f 0 1517854903119 3 connected
bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002 master - 0 1517854903621 3 connected 12182-16383
54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006 myself,master - 0 0 8 connected 0-1364 5461-6826 10923-12181 [12182-<-bb31a89640b381e8de18f15a845512f08ab9f16f]
0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001 master - 0 1517854903621 2 connected 6827-10922
a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000 slave df9ac983886f0f09b0c62feef9288a9296a1a08c 0 1517854902617 7 connected
127.0.0.1:7006> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:7
cluster_size:4
cluster_current_epoch:8
cluster_my_epoch:8
cluster_stats_messages_sent:7375
cluster_stats_messages_received:7362

127.0.0.1:7001> set age 100
-> Redirected to slot [741] located at 127.0.0.1:7006
OK
127.0.0.1:7006> get age
"100"

````

暂时集群时可以用的,不知道以上的error有啥影响.

### 新建一个7007从节点

新建一个节点,启动和之前类似,不做描述.

加入一个新节点,7007 

````
zhuningning@ubuntu:/usr/local/redisCluster/7007$ ./src/redis-trib.rb add-node --slave 127.0.0.1:7007 127.0.0.1:7000
>>> Adding node 127.0.0.1:7007 to cluster 127.0.0.1:7000
>>> Performing Cluster Check (using node 127.0.0.1:7000)
S: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots: (0 slots) slave
   replicates df9ac983886f0f09b0c62feef9288a9296a1a08c
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
M: 54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006
   slots:0-1364,5461-6826,10923-12181 (3990 slots) master
   0 additional replica(s)
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:12182-16383 (4202 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
[WARNING] Node 127.0.0.1:7006 has slots in importing state (12182).
[WARNING] Node 127.0.0.1:7002 has slots in migrating state (12182).
[WARNING] The following slots are open: 12182
>>> Check slots coverage...
[OK] All 16384 slots covered.
Automatically selected master 127.0.0.1:7006
>>> Send CLUSTER MEET to node 127.0.0.1:7007 to make it join the cluster.
Waiting for the cluster to join.20427:M 06 Feb 02:30:11.160 # IP address for this node updated to 127.0.0.1

>>> Configure node as replica of 127.0.0.1:7006.
20427:S 06 Feb 02:30:12.032 # Cluster state changed: ok
[OK] New node added correctly.
zhuningning@ubuntu:/usr/local/redisCluster/7007$ 20427:S 06 Feb 02:30:12.136 * Connecting to MASTER 127.0.0.1:7006
20427:S 06 Feb 02:30:12.136 * MASTER <-> SLAVE sync started
20427:S 06 Feb 02:30:12.136 * Non blocking connect for SYNC fired the event.
20427:S 06 Feb 02:30:12.136 * Master replied to PING, replication can continue...
20427:S 06 Feb 02:30:12.137 * Partial resynchronization not possible (no cached master)
15049:M 06 Feb 02:30:12.137 * Slave 127.0.0.1:7007 asks for synchronization
15049:M 06 Feb 02:30:12.137 * Full resync requested by slave 127.0.0.1:7007
15049:M 06 Feb 02:30:12.137 * Starting BGSAVE for SYNC with target: disk
15049:M 06 Feb 02:30:12.138 * Background saving started by pid 20906
20427:S 06 Feb 02:30:12.139 * Full resync from master: defd3e21a2f15f27e3319035074c5002d4ff4b1b:1
20906:C 06 Feb 02:30:12.500 * DB saved on disk
20906:C 06 Feb 02:30:12.501 * RDB: 6 MB of memory used by copy-on-write
15049:M 06 Feb 02:30:12.509 * Background saving terminated with success
20427:S 06 Feb 02:30:12.510 * MASTER <-> SLAVE sync: receiving 89 bytes from master
15049:M 06 Feb 02:30:12.510 * Synchronization with slave 127.0.0.1:7007 succeeded
20427:S 06 Feb 02:30:12.510 * MASTER <-> SLAVE sync: Flushing old data
20427:S 06 Feb 02:30:12.510 * MASTER <-> SLAVE sync: Loading DB in memory
20427:S 06 Feb 02:30:12.510 * MASTER <-> SLAVE sync: Finished with success
20427:S 06 Feb 02:30:12.512 * Background append only file rewriting started by pid 20907
20427:S 06 Feb 02:30:12.581 * AOF rewrite child asks to stop sending diffs.
20907:C 06 Feb 02:30:12.582 * Parent agreed to stop sending diffs. Finalizing AOF...
20907:C 06 Feb 02:30:12.582 * Concatenating 0.00 MB of AOF diff received from parent.
20907:C 06 Feb 02:30:12.582 * SYNC append only file rewrite performed
20907:C 06 Feb 02:30:12.583 * AOF rewrite: 6 MB of memory used by copy-on-write
20427:S 06 Feb 02:30:12.639 * Background AOF rewrite terminated with success
20427:S 06 Feb 02:30:12.639 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
20427:S 06 Feb 02:30:12.639 * Background AOF rewrite finished successfully

````

add-node的时候加上--slave表示是加入到从节点中，但是这样加，是随机的。这里的命令行完全像我们在添加一个新主服务器时使用的一样，所以我们没有指定要给哪个主服 务器添加副本。这种情况下，redis-trib 会将7007作为一个具有较少副本的随机的主服务器的副本。

- 加入从节点,为该从节点指定一个主节点 

新建7008节点

````

zhuningning@ubuntu:/usr/local/redisCluster/7008$ ./src/redis-trib.rb add-node --slave --master-id df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7008 127.0.0.1:7000
>>> Adding node 127.0.0.1:7008 to cluster 127.0.0.1:7000
>>> Performing Cluster Check (using node 127.0.0.1:7000)
S: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots: (0 slots) slave
   replicates df9ac983886f0f09b0c62feef9288a9296a1a08c
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
M: df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
S: db439109563a804d8d63ffdb6937baf88adde5cc 127.0.0.1:7007
   slots: (0 slots) slave
   replicates 54fe323634d8830f2ef93feaaedb2b70e19d0765
M: 54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006
   slots:0-1364,5461-6826,10923-12181 (3990 slots) master
   1 additional replica(s)
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:12182-16383 (4202 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
[WARNING] Node 127.0.0.1:7006 has slots in importing state (12182).
[WARNING] Node 127.0.0.1:7002 has slots in migrating state (12182).
[WARNING] The following slots are open: 12182
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 127.0.0.1:7008 to make it join the cluster.
Waiting for the cluster to join.21391:M 06 Feb 02:36:05.351 # IP address for this node updated to 127.0.0.1

>>> Configure node as replica of 127.0.0.1:7003.
21391:S 06 Feb 02:36:06.188 # Cluster state changed: ok
[OK] New node added correctly.
zhuningning@ubuntu:/usr/local/redisCluster/7008$ 21391:S 06 Feb 02:36:06.612 * Connecting to MASTER 127.0.0.1:7003
21391:S 06 Feb 02:36:06.612 * MASTER <-> SLAVE sync started
21391:S 06 Feb 02:36:06.612 * Non blocking connect for SYNC fired the event.
21391:S 06 Feb 02:36:06.613 * Master replied to PING, replication can continue...
21391:S 06 Feb 02:36:06.613 * Partial resynchronization not possible (no cached master)
4701:M 06 Feb 02:36:06.613 * Slave 127.0.0.1:7008 asks for synchronization
4701:M 06 Feb 02:36:06.614 * Full resync requested by slave 127.0.0.1:7008
4701:M 06 Feb 02:36:06.614 * Starting BGSAVE for SYNC with target: disk
4701:M 06 Feb 02:36:06.616 * Background saving started by pid 21659
21391:S 06 Feb 02:36:06.617 * Full resync from master: cecbe45078fac08152e9ca95d7741ae086b4ea68:4201
21659:C 06 Feb 02:36:06.659 * DB saved on disk
21659:C 06 Feb 02:36:06.660 * RDB: 8 MB of memory used by copy-on-write
4701:M 06 Feb 02:36:06.753 * Background saving terminated with success
4701:M 06 Feb 02:36:06.753 * Synchronization with slave 127.0.0.1:7008 succeeded
21391:S 06 Feb 02:36:06.753 * MASTER <-> SLAVE sync: receiving 77 bytes from master
21391:S 06 Feb 02:36:06.754 * MASTER <-> SLAVE sync: Flushing old data
21391:S 06 Feb 02:36:06.754 * MASTER <-> SLAVE sync: Loading DB in memory
21391:S 06 Feb 02:36:06.754 * MASTER <-> SLAVE sync: Finished with success
21391:S 06 Feb 02:36:06.755 * Background append only file rewriting started by pid 21660
21391:S 06 Feb 02:36:06.790 * AOF rewrite child asks to stop sending diffs.
21660:C 06 Feb 02:36:06.790 * Parent agreed to stop sending diffs. Finalizing AOF...
21660:C 06 Feb 02:36:06.791 * Concatenating 0.00 MB of AOF diff received from parent.
21660:C 06 Feb 02:36:06.791 * SYNC append only file rewrite performed
21660:C 06 Feb 02:36:06.792 * AOF rewrite: 6 MB of memory used by copy-on-write
21391:S 06 Feb 02:36:06.814 * Background AOF rewrite terminated with success
21391:S 06 Feb 02:36:06.814 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
21391:S 06 Feb 02:36:06.814 * Background AOF rewrite finished successfully

````

--master-id 指定7003节点成为7008节点的主节点

````
zhuningning@ubuntu:/usr/local/redisCluster/7008$ ./src/redis-cli -c -p 7008 
127.0.0.1:7008> cluster nodes
db439109563a804d8d63ffdb6937baf88adde5cc 127.0.0.1:7007 slave 54fe323634d8830f2ef93feaaedb2b70e19d0765 0 1517855860750 8 connected
bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002 master - 0 1517855861752 3 connected 12182-16383
54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006 master - 0 1517855861751 8 connected 0-1364 5461-6826 10923-12181
df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003 master - 0 1517855859747 7 connected 1365-5460
0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001 master - 0 1517855860248 2 connected 6827-10922
37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004 slave 0ca3673dfd352052b651b3898ba3f807ff9c7f55 0 1517855861251 2 connected
442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005 slave bb31a89640b381e8de18f15a845512f08ab9f16f 0 1517855860750 3 connected
c29ea305fc8283a9987629b10dbc2ccae97c0c38 127.0.0.1:7008 myself,slave df9ac983886f0f09b0c62feef9288a9296a1a08c 0 0 0 connected
a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000 slave df9ac983886f0f09b0c62feef9288a9296a1a08c 0 1517855860248 7 connected

````


7003是7008和7000的主节点,kill掉70003,7000成为主节点

````
127.0.0.1:7008> cluster nodes
db439109563a804d8d63ffdb6937baf88adde5cc 127.0.0.1:7007 slave 54fe323634d8830f2ef93feaaedb2b70e19d0765 0 1517856011263 8 connected
bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002 master - 0 1517856010761 3 connected 12182-16383
54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006 master - 0 1517856010862 8 connected 0-1364 5461-6826 10923-12181
df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003 master,fail - 1517855992209 1517855990103 7 disconnected
0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001 master - 0 1517856009257 2 connected 6827-10922
37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004 slave 0ca3673dfd352052b651b3898ba3f807ff9c7f55 0 1517856010761 2 connected
442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005 slave bb31a89640b381e8de18f15a845512f08ab9f16f 0 1517856009757 3 connected
c29ea305fc8283a9987629b10dbc2ccae97c0c38 127.0.0.1:7008 myself,slave a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 0 0 0 connected
a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000 master - 0 1517856010258 9 connected 1365-5460

````


### 移除一个主节点

和新加节点有点不同的是，移除需要节点的node-id。那我们尝试将7002这个主节点移除：

````
zhuningning@ubuntu:/usr/local/redisCluster/7008$ ./src/redis-trib.rb del-node 127.0.0.1:7000 bb31a89640b381e8de18f15a845512f08ab9f16f
>>> Removing node bb31a89640b381e8de18f15a845512f08ab9f16f from cluster 127.0.0.1:7000
[ERR] Node 127.0.0.1:7002 is not empty! Reshard data away and try again.

````
提示节点非空,需要重新分片

````
zhuningning@ubuntu:/usr/local/redisCluster/7008$ ./src/redis-trib.rb reshard 127.0.0.1:7000
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
S: db439109563a804d8d63ffdb6937baf88adde5cc 127.0.0.1:7007
   slots: (0 slots) slave
   replicates 54fe323634d8830f2ef93feaaedb2b70e19d0765
M: 54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006
   slots:0-1364,5461-6826,10923-12181 (3990 slots) master
   1 additional replica(s)
M: 0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
S: c29ea305fc8283a9987629b10dbc2ccae97c0c38 127.0.0.1:7008
   slots: (0 slots) slave
   replicates a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4
S: 37b5d56cbce25be2e3ad25b52ce39e3e4885654e 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 0ca3673dfd352052b651b3898ba3f807ff9c7f55
S: 442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005
   slots: (0 slots) slave
   replicates bb31a89640b381e8de18f15a845512f08ab9f16f
M: bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002
   slots:12182-16383 (4202 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
[WARNING] Node 127.0.0.1:7006 has slots in importing state (12182).
[WARNING] Node 127.0.0.1:7002 has slots in migrating state (12182).
[WARNING] The following slots are open: 12182
>>> Check slots coverage...
[OK] All 16384 slots covered.
*** Please fix your cluster problems before resharding

````

由于上面的error,现在不能重新分片,所以暂时没办法演示删除主节点.如果你的可以重新分片,则将7002中的槽位移动到其它某个master节点(不知道slave节点是否可以,读者可以自行实验).

然后重新执行删除操作即可.

### 移除一个从节点

由于不考虑数据迁移,则直接删除即可,在此删除从节点7004

````
zhuningning@ubuntu:/usr/local/redisCluster/7008$ ./src/redis-trib.rb del-node 127.0.0.1:7000 37b5d56cbce25be2e3ad25b52ce39e3e4885654e
>>> Removing node 37b5d56cbce25be2e3ad25b52ce39e3e4885654e from cluster 127.0.0.1:7000
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
4772:S 06 Feb 02:52:42.770 # User requested shutdown...
4772:S 06 Feb 02:52:42.770 * Calling fsync() on the AOF file.
4772:S 06 Feb 02:52:42.770 * Saving the final RDB snapshot before exiting.
4772:S 06 Feb 02:52:42.860 * DB saved on disk
4772:S 06 Feb 02:52:42.860 * Removing the pid file.
4772:S 06 Feb 02:52:42.860 # Redis is now ready to exit, bye bye...
4741:M 06 Feb 02:52:42.861 # Connection with slave 127.0.0.1:7004 lost.
[5]   Done                    sudo ./src/redis-server redis.conf  (wd: /usr/local/redisCluster/7004)
(wd now: /usr/local/redisCluster/7008)

````
查询集群状态,被删除的和fail状态的不同.fail仍然数据该集群,恢复后仍然可以加入集群,删除则不存在集群了,需要手动操作去加入集群.

````
127.0.0.1:7008> cluster nodes
db439109563a804d8d63ffdb6937baf88adde5cc 127.0.0.1:7007 slave 54fe323634d8830f2ef93feaaedb2b70e19d0765 0 1517856814381 8 connected
bb31a89640b381e8de18f15a845512f08ab9f16f 127.0.0.1:7002 master - 0 1517856814884 3 connected 12182-16383
54fe323634d8830f2ef93feaaedb2b70e19d0765 127.0.0.1:7006 master - 0 1517856814884 8 connected 0-1364 5461-6826 10923-12181
df9ac983886f0f09b0c62feef9288a9296a1a08c 127.0.0.1:7003 master,fail - 1517855992209 1517855990103 7 disconnected
0ca3673dfd352052b651b3898ba3f807ff9c7f55 127.0.0.1:7001 master - 0 1517856815383 2 connected 6827-10922
442c6654dfda38fe6544c96627fcafaac833a7be 127.0.0.1:7005 slave bb31a89640b381e8de18f15a845512f08ab9f16f 0 1517856814381 3 connected
c29ea305fc8283a9987629b10dbc2ccae97c0c38 127.0.0.1:7008 myself,slave a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 0 0 0 connected
a05c8dcbd5e7861000b3b5bf7e28d6843c85efe4 127.0.0.1:7000 master - 0 1517856813375 9 connected 1365-5460

````

[redis集群研究和实践](https://www.zybuluo.com/phper/note/195558)

[在线迁移redis cluster](http://blog.51cto.com/hsbxxl/1978491)



















