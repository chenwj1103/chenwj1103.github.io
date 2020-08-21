---
title: redis设计与实现-cluster
copyright: true
date: 2018-01-28 00:18:54
tags: redis cluster
categories: redis

---

Redis集群时redis提供的分布式数据库方案.

# 节点

一个 Redis 集群通常由多个节点（node）组成， 在刚开始的时候， 每个节点都是相互独立的， 它们都处于一个只包含自己的集群当中， 要组建一个真正可工作的集群， 我们必须将各个独立的节点连接起来， 构成一个包含多个节点的集群。

当一个节点使用下面的命令时,使得ip和port制定的节点进行握手,当捂手成功后,node节点就会将该ip和port所在的节点添加到自己的集群中.

````
CLUSTER MEET <ip> <port>

````
假设我们当前有三个节点:
````
root     21254  9329  0 01:35 pts/2    00:00:00 sudo ./src/redis-server redis.conf
root     21255 21254  0 01:35 pts/2    00:00:00 ./src/redis-server 127.0.0.1:6002 [cluster]
root     21263  9329  0 01:36 pts/2    00:00:00 sudo ./src/redis-server redis.conf
root     21264 21263  0 01:36 pts/2    00:00:00 ./src/redis-server 127.0.0.1:6001 [cluster]
root     21283  9329  0 01:36 pts/2    00:00:00 sudo ./src/redis-server redis.conf
root     21284 21283  0 01:36 pts/2    00:00:00 ./src/redis-server 127.0.0.1:6000 [cluster]
zhuning+ 21466  9329  0 01:37 pts/2    00:00:00 grep --color=auto redis

````
将节点加入集群

````
zhuningning@ubuntu:/usr/local/redisClusterTest/6000/src$ ./redis-cli -p 127.0.0.1 -p 6000
127.0.0.1:6000> cluster nodes
e6f7de12f623d0a92da8aa307a7ee224da73b92e :6000 myself,master - 0 0 0 connected
127.0.0.1:6000> cluster meet 127.0.0.1 6001
OK
127.0.0.1:6000> 21264:M 29 Jan 01:39:45.936 # IP address for this node updated to 127.0.0.1
                                                                                           21284:M 29 Jan 01:39:45.975 # IP address for this node updated to 127.0.0.1 
127.0.0.1:6000> cluster nodes
5e57d9227987c64486ec791217b64712ad201016 127.0.0.1:6001 master - 0 1517161211620 1 connected
e6f7de12f623d0a92da8aa307a7ee224da73b92e 127.0.0.1:6000 myself,master - 0 0 0 connected
127.0.0.1:6000> cluster meet 127.0.0.1 6002
OK
127.0.0.1:6000> 21255:M 29 Jan 01:41:29.250 # IP address for this node updated to 127.0.0.1
127.0.0.1:6000> 
127.0.0.1:6000> clsuter nodes
(error) ERR unknown command 'clsuter'
127.0.0.1:6000> cluster nodes
94a98e405ec368ad45469b9d5fb0e019bdf2d796 127.0.0.1:6002 master - 0 1517161300885 2 connected
5e57d9227987c64486ec791217b64712ad201016 127.0.0.1:6001 master - 0 1517161299882 1 connected
e6f7de12f623d0a92da8aa307a7ee224da73b92e 127.0.0.1:6000 myself,master - 0 0 0 connected

````

以上是建立集群的过程

## 启动节点

启动的时候redis会根据参数cluster-enabled参数来决定是否启动集群模式.

node在集群模式下仍然会使用单机集群的功能,比如运行serverCron函数,而serverCron函数又会调用clusterCron函数.

除此之外节点会继续使用 redisServer 结构来保存服务器的状态， 使用 redisClient 结构来保存客户端的状态， 至于那些只有在集群模式下才会用到的数据， 节点将它们保存到了 cluster.h/clusterNode 结构， cluster.h/clusterLink 结构， 以及 cluster.h/clusterState 结构里面

## 集群数据结构

每个节点都会使用一个 clusterNode 结构来记录自己的状态， 并为集群中的所有其他节点（包括主节点和从节点）都创建一个相应的 clusterNode 结构

结构如下:

````
struct clusterNode {

    // 创建节点的时间
    mstime_t ctime;

    // 节点的名字，由 40 个十六进制字符组成
    // 例如 68eef66df23420a5862208ef5b1a7005b806f2ff
    char name[REDIS_CLUSTER_NAMELEN];

    // 节点标识
    // 使用各种不同的标识值记录节点的角色（比如主节点或者从节点），
    // 以及节点目前所处的状态（比如在线或者下线）。
    int flags;

    // 节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch;

    // 节点的 IP 地址
    char ip[REDIS_IP_STR_LEN];

    // 节点的端口号
    int port;

    // 保存连接节点所需的有关信息
    clusterLink *link;

    // ...

};

````

clusterNode 结构的 link 属性是一个 clusterLink 结构， 该结构保存了连接节点所需的有关信息， 比如套接字描述符， 输入缓冲区和输出缓冲区：

````
typedef struct clusterLink {

    // 连接的创建时间
    mstime_t ctime;

    // TCP 套接字描述符
    int fd;

    // 输出缓冲区，保存着等待发送给其他节点的消息（message）。
    sds sndbuf;

    // 输入缓冲区，保存着从其他节点接收到的消息。
    sds rcvbuf;

    // 与这个连接相关联的节点，如果没有的话就为 NULL
    struct clusterNode *node;

} clusterLink;


````

- redisClient 结构和 clusterLink 结构的相同和不同之处

redisClient 结构和 clusterLink 结构都有自己的套接字描述符和输入、输出缓冲区， 这两个结构的区别在于， redisClient 结构中的套接字和缓冲区是用于连接客户端的， 而 clusterLink 结构中的套接字和缓冲区则是用于连接节点的。


每个节点都保存着一个 clusterState 结构， 这个结构记录了在当前节点的视角下， 集群目前所处的状态 —— 比如集群是在线还是下线， 集群包含多少个节点， 集群当前的配置纪元

````
typedef struct clusterState {

    // 指向当前节点的指针
    clusterNode *myself;

    // 集群当前的配置纪元，用于实现故障转移
    uint64_t currentEpoch;

    // 集群当前的状态：是在线还是下线
    int state;

    // 集群中至少处理着一个槽的节点的数量
    int size;

    // 集群节点名单（包括 myself 节点）
    // 字典的键为节点的名字，字典的值为节点对应的 clusterNode 结构
    dict *nodes;

    // ...

} clusterState;

````

创建了三个节点的集群结构如下图:

![创建了三个节点的集群结构](/images/redis/cluster/创建了三个节点的集群结构.png)

结构的 currentEpoch 属性的值为 0 ， 表示集群当前的配置纪元为 0 。

结构的 size 属性的值为 0 ， 表示集群目前没有任何节点在处理槽： 因此结构的 state 属性的值为 REDIS_CLUSTER_FAIL —— 这表示集群目前处于下线状态。

结构的 nodes 字典记录了集群目前包含的三个节点， 这三个节点分别由三个 clusterNode 结构表示： 其中 myself 指针指向代表节点 7000 的 clusterNode 结构， 而字典中的另外两个指针则分别指向代表节点 7001 和代表节点 7002 的 clusterNode 结构， 这两个节点是节点 7000 已知的在集群中的其他节点。

三个节点的 clusterNode 结构的 flags 属性都是 REDIS_NODE_MASTER ，说明三个节点都是主节点。

## CLUSTER MEET 命令的实现¶

````
CLUSTER MEET <ip> <port>
````
节点A发送以上的命令给B.A节点与B节点进行握手:

1. 节点 A 会为节点 B 创建一个 clusterNode 结构， 并将该结构添加到自己的 clusterState.nodes 字典里面。
2. 之后， 节点 A 将根据 CLUSTER MEET 命令给定的 IP 地址和端口号， 向节点 B 发送一条 MEET 消息（message）。
3. 如果一切顺利， 节点 B 将接收到节点 A 发送的 MEET 消息， 节点 B 会为节点 A 创建一个 clusterNode 结构， 并将该结构添加到自己的 clusterState.nodes 字典里面。
4. 之后， 节点 B 将向节点 A 返回一条 PONG 消息。
5. 如果一切顺利， 节点 A 将接收到节点 B 返回的 PONG 消息， 通过这条 PONG 消息节点 A 可以知道节点 B 已经成功地接收到了自己发送的 MEET 消息。
6. 之后， 节点 A 将向节点 B 返回一条 PING 消息。
7. 如果一切顺利， 节点 B 将接收到节点 A 返回的 PING 消息， 通过这条 PING 消息节点 B 可以知道节点 A 已经成功地接收到了自己返回的 PONG 消息， 握手完成。

# 槽指派

数据库中的每个键都属于16384个槽位中的其中一个,集群中的每个及诶点都可以处理0个最多16384个槽位.之后全部的槽为被处理后,那么这个集群才处于上线状态,否则处于下线状态.

上一节中,cluster meet命令将三个节点加到一个集群中,但是没有进行槽位指派,所以集群处于fail状态.

````
zhuningning@ubuntu:/usr/local/redisClusterTest/6002$ ./src/redis-cli -h 127.0.0.1 -p 6002
127.0.0.1:6002> 
127.0.0.1:6002> 
127.0.0.1:6002> cluster info
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:0
cluster_current_epoch:2
cluster_my_epoch:2
cluster_stats_messages_sent:1858
cluster_stats_messages_received:0

````

这样启动后,查看集群状态,处于fail状态.

可以连接上连接上node节点使用 
````
127.0.0.1:6000> cluster addslots 0 1 2 3 ... 5000
127.0.0.1:6001> cluster addslots 5001 5002 2 3 ... 10000
127.0.0.1:6002> cluster addslots 10001 10002 10003 ... 16383 
````
分配槽位.

````
127.0.0.1:6000> cluster nodes
5e57d9227987c64486ec791217b64712ad201016 127.0.0.1:6001 master - 0 1517243097362 1 connected
94a98e405ec368ad45469b9d5fb0e019bdf2d796 127.0.0.1:6002 master - 0 1517243098365 2 connected
e6f7de12f623d0a92da8aa307a7ee224da73b92e 127.0.0.1:6000 myself,master - 0 0 0 connected

````

## 记录节点的槽指派信息

clusterNode结构的slots和numslot属性记录了节点负责处理哪些槽.

![clusterMode结构](/images/redis/cluster/clusterMode结构.png)

slots属性时一个二进制位数组.这个数组的长度时16384/8=2048个字节,共包含16384个二进制位.

![slots数组示例](/images/redis/cluster/slots数组示例.png)

值是1表示该节点处理该槽位.

由于取出和设置slots数组中的任意一个二进制位的值的复杂度仅为O(1),所以对于一个制定节点的slots数组来说,检查该节点是否处理某个槽,时间复杂度都为O(1).

## 传播节点的槽指派信息

一个节点除了会将自己负责的槽记录在clusterNode结构的slots属性和numslots属性中,还会将自己的slots数组通过消息发送到集群中的其他节点,以此来告知其他节点自己目前处理的哪些槽.

![node告知别的节点自己处理的槽位](/images/redis/cluster/node告知别的节点自己处理的槽位.png)

这样,某个节点会将别的节点处理的槽位记录在其记录的该节点的clusterNode结构中.

## 记录集群所有槽的指派信息

clusterState结构的slots数组记录了集群中所有的槽位的指派信息.如果只将槽位的指派信息记录到clusterNode节点中,则查找某个槽位在哪个节点还要遍历各个节点,时间复杂度为O(N)

有了clusterState结构,则时间复杂度为O(1)

![clusterState结构的slots数组](/images/redis/cluster/clusterState结构的slots数组.png)

clusterState结构的slots数组包含16384个项,存储的是指向某一个clusterNode节点的指针.

clusterNode.slots数组记录了节点的指派信息,clusterState.slots数组记录了所有的槽位指派信息.前者是便于发送槽位指派信息,后者是便于查找某一个槽位在哪个节点.

## CLUSTER ADDSLOTS命令的实现

cluster addslots<slot>[slot ...] ,将槽位指派给接收该命令的节点负责

![cluster addslots 的伪代码实现](/images/redis/cluster/clusteraddslots的伪代码实现.png)

未指派的cluster结构:

![未指派的cluster结构](/images/redis/cluster/未指派的cluster结构.png)

执行命令后
````
cluster addslots 1 2
````
![执行cluster addSlots命令后](/images/redis/cluster/执行cluster addSlots命令后.png)


之后,节点会通过发送命令发送消息告知集群中的其它节点,自己正在处理的哪些槽.

# 在集群中执行命令

槽位指派完后,集群进入上线状态.这时客户端就可以向集群发送数据命令了.

![判断客客户端是否需要转向其它流程](/images/redis/cluster/判断客客户端是否需要转向其它流程.png)

## 计算键属于哪个槽

计算键属于哪个槽位

![计算键的槽位](/images/redis/cluster/计算键的槽位.png)

cluster keyslot <key> 查看给定的键属于哪个槽.

## 判断槽位是否由当前节点处理

当节点计算出键所属的槽i后,接待节点检查自己在clusterState.slots数组的项i,看槽是否由自己负责.

如果自己负责,则执行命令,否则返回负责该槽位的节点ip和端口号.并向客户端返回 moved错误.

## moved错误

MOVED错误的格式:

moved<slot><ip>:<port> ,其中slot是键所在的槽,ip和por是处理该键的节点.

一个集群客户端通常会与集群中的多个节点建立套接字连接.而所谓的节点转向其实就是换一个节点自来发送命令.

单机模式的客户端会打印moved信息,而集群客户端不会打印,只是自动进行节点转向.

## 节点数据库的实现

节点数据库只可以使用0号数据库.**节点使用clusterState结构中的slots_to_keys跳跃表来保存槽和键之间的关系.**

slots_to_keys跳跃表每个节点的分之都是一个槽号,每个节点的成员都是一个数据库键.

当往数据库中添加一个新的键值对时,节点都会将这个键及键的槽号关联到slots_to_keys跳跃表.

![节点7000的跳跃表](/images/redis/cluster/节点7000的跳跃表.png)

# 重新分片

redis集群的重新分片操作可以将任意数量已经指派给某个几点的槽改为指派给另一个节点,并且想过槽所属的键值对也会从源节点移动到目标节点.

重新分片可以在线进行,在重新分片的过程中,集群不需要下线,并且源节点和目标节点都可以继续处理命令请求.

## 重新分片的实现原理:

redis集群的重新分片操作是有redis的集群管理软件redis-trib负责执行的.redis-trib通过向源节点和目标节点发送命令来实现的.

![重新分片的操作](/images/redis/cluster/重新分片的操作.png)


![迁移键的过程](/images/redis/cluster/迁移键的过程.png)

![对槽进行重新分片的过程](/images/redis/cluster/对槽进行重新分片的过程.png)

# ASK错误

在进行重新分片期间,源节点向目标节点迁移一个槽的过程中.可能出现:属于被迁移槽的一部分键值对保存在源节点里面,而另一部分键值对保存在目标节点里面.

当客户端向源节点发送一个与数据库键有关的命令,并且命令要处理的数据恰好属于被迁移的槽时:

源节点会先在自己的数据库里面查找制定的键,如果找到的话,就直接执行客户端发送的命令.源节点没能在自己的数据库里面找到制定的键,那么键有可能被迁移到目标节点,源节点向客户端返回一个ASK错误,指引客户端转向正在导入槽的目标节点,并再次发送之前要执行的命令.

![判断是否发送ASK错误的过程](/images/redis/cluster/判断是否发送ASK错误的过程.png)

和moved错误情况类似,集群模式的客户端会自动根据错误提供的ip和端口进行转向动作.

## cluster setslot importing命令的实现

clusterState结构的importing_slots_from数组记录了当前节点正在从其他节点导入的槽

![importing_slots_from数组](/images/redis/cluster/importing_slots_from数组.png)

如果importing_slots_from[i]不为null,而是指向一个clusterNode结构,那么表示当前节点正在从clusterNode所代表的节点导入槽.

在对集群重新进行分片的时候,向目标节点发送命令:

cluster setslot<i> importing <source_id> ,可以将目标节点的clusterState.importing_slots_from[i] 的值设置为source_id所代表的cluster_node结构

![发送setslot命令](/images/redis/cluster/发送setslot命令.png)

## cluster setslot migrating命令的实现

clusterState结构的migrating_slots_to数组记录了当前节点正在迁移至其他节点的槽.

如果migrating_slots_to[i]不为null,而是指向一个clusterNode结构,那么表示当前节点正在将槽i迁移至clusterNode所代表的节点.

节点7002的migrating_slots_to数组:

![节点7002的migrating_slots_to数组](/images/redis/cluster/节点7002的migrating_slots_to数组.png)

## ASK错误

如果节点收到一个关于键key的命令请求,并且键key所属的槽i正好指派给该节点,则该节点处理该命令.否则节点会检查migrating_slots_to[i],看键key所属的槽i是否正在进行迁移,如果槽i的确正在迁移,那么节点会向客户端返回一个ASK错误.

比如返回 ASK 16198 127.0.0.1:7003 表示客户端可以尝试到ip为127.0.0.1,端口号为7003的节点去执行和槽16198有关的操作.

接收到ASK错误的客户端会根据错误提供的ip和端口号,转向正在导入槽的目标及节点,然后首先向目标节点发送一个ASKING命令,之后重新发送原本想要执行的命令.

## ASKING命令

ASKING命令唯一要做的就是打开发送该命令的客户端redis_asking标示.

![节点判断是否执行客户端命令的过程](/images/redis/cluster/节点判断是否执行客户端命令的过程.png)

当客户端接收到ASK错误并转向至正在导入槽的节点时,客户端会首先发送ASKING命令,然后才重新发送需要执行的命令.如果不发送ASKING命令,直接发送要执行的命令,客户端会拒绝执行,返回moved错误.

客户端redis_asking标示是一次性的.当节点执行了一个带有该标识的命令后,该标示就会被移除.

## ASK错误和MOVED错误的区别

![ASK错误和MOVED错误的区别](/images/redis/cluster/ASK错误和MOVED错误的区别.png)

# 复制和故障转移

redis集群中的节点分为主节点和从节点,主节点用于处理槽,从节点用于复制某个主节点,并在被复制的主节点下线时,代替下线朱及诶点继续处理命令请求.

## 设置从节点

向一个节点发送
````
cluster replicate<node_id>
````
可以让接收命令的节点成为node_id所指定节点的从节点,并开始对主节点进行复制.

1. 接收到命令的节点首先从clusterState.nodes字典中找到node_id所对应节点的clusterNode结构,,并将自己的clusterState.mysql.slaveof指针指向这个结构,以此来记录这个节点正在复制主节点.
2. 然后节点会修改自己在clusterState.myself.flags中的属性.关闭原本的REDIS_NODE_MASTER属性.打开redis_node_slave属性.
3. 最后,节点会调用复制代码,并根据clusterState.myself.slaveof指向的clusterNode结构保存的ip地址和端口号.

![7004从节点的clusterState结构](/images/redis/cluster/7004从节点的clusterState结构.png)

## 故障检测

集群中的每个节点都会定时的向集群中的其它节点发送ping消息,以此来检测对方是否在线;如果接收ping消息的节点在规定的时间内,回复pong消息,那么发送ping消息的及诶点就会向接收ping消息的节点标记为疑似下线状态(PFAIL);

集群中的各个节点都会通过互相发送消息的方式来交换集群中各个节点的状态消息,例如某个节点处于下线状态还是疑似下线状态.

当一个主节点A通过消息得知主节点B认为主节点C进入了疑似下线状态时,主节点A会在自己的clusterState.nodes字典中找到主节点C对应的clusterNode结构,并将主节点B的下线报告添加到clusterNode结构的fail_repoorts链表里面.

当主节点7001在收到主节点7002/主节点7003发送的消息后得知,主节点7002和主节点7003都认为主节点7000进入了疑似下线状态,那么主节点7001将为主节点7000创建如下的下线报告

![节点7000的下线报告](/images/redis/cluster/节点7000的下线报告.png)

如果一个集群里面,半数以上的负责处理槽的主节点将某个主节点X报告为疑似下线,那么主节点X将被将被标记为下线状态,将主节点X标记为下线状态的节点会向集群广播一条关于主节点X的fail消息,那么收到这条Fail消息的节点将会立即将主节点X标记为已下线.

## 故障转移

当从节点发现主节点进入下线状态,从节点开始对主节点进行故障转移:

![故障转移的步骤](/images/redis/cluster/故障转移的步骤.png)

## 选举新的主节点

基于Raft算法:

![选取新的主节点1](/images/redis/cluster/选取新的主节点1.png)

![选取新的主节点2](/images/redis/cluster/选取新的主节点2.png)


# 消息

- meet消息

发送者请求接收该消息者加入到当前所在的集群里面

- ping消息

集群中每个节点默认每隔一秒钟都会从已知节点列表中随机选出5隔节点发送ping消息,一次来检验被选中的节点是否在线.

- pong消息

接收ping或者meet消息的接受者会发送pong来回复发送者.还有在故障转移后,新的主节点会向集群广播一条pong消息,以此让集群知道该节点成为主节点,并接管了已经下线节点负责的槽.

- fail消息

当A节点判断B节点进入Fail状态,它会向集群广播一条关于节点B的Fail消息,所有收到消息的节点会将B标记为下线状态.

- publish消息

当节点接收到一个PUBLISH命令时,节点会向集群广播一条PUBLISH消息,所有接收到这条PUBLISH消息的节点都会执行相同的PUBLISH命令.

## 消息头

节点发送的消息都由一个消息头包裹,消息头除了包含消息正文之外,还记录了消息发送者自身的一些信息.
































