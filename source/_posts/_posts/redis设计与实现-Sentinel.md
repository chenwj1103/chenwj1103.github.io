---
title: redis设计与实现-Sentinel
copyright: true
date: 2018-01-26 00:02:28
tags: Sentinel,redis
categories: redis

---

Sentinel是redis的高可用性(High availability)的解决方案.有一个或者多个Sentinel的实例组成的Sentinel系统可以监视任意多个主服务器以及这些主服务器下的从服务器.当主服务下线的时候,从服务器会成为主服务器来执行主服务器的任务.

双环代表主服务器,单环代表从服务器

![服务器以Sentinel系统](/images/redis/sentinel/服务器以Sentinel系统.png)

在主服务下线的时候,三个从服务器停止对主服务器的复制.

![主服务器下线](/images/redis/sentinel/主服务器下线.png)

当server1的下线时间超过用户设定的下线时长上限时,sentinel系统会对server1进行故障转移.

1. 首先sentinel系统挑选server1属下的其中一个从服务器,并将这个被选中的从服务器升级为新的主服务器.
2. 之后,sentinel系统会向server1属下的所有从服务器发送新的复制指令,让他们成为新的主服务器的从服务器.当所有的从服务器都开始复制新的主服务器时,故障转移操作执行完毕.
3. 另外,sentinel会继续监视server1,并在它重新上线时,将他设置为新的主服务器的从服务器.

![故障转移以及原来的服务器恢复](/images/redis/sentinel/故障转移以及原来的服务器恢复.png)

# 启动并初始化Sentinel

````
zhuningning@ubuntu:/usr/local/redis-3.2.10/src$ ./redis-sentinel ../sentinel.conf &
[2] 13772
zhuningning@ubuntu:/usr/local/redis-3.2.10/src$ 13772:X 27 Jan 01:05:25.512 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.2.10 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 26379
 |    `-._   `._    /     _.-'    |     PID: 13772
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

13772:X 27 Jan 01:05:25.514 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
13772:X 27 Jan 01:05:25.514 # Sentinel ID is 9aa0e1e3a36ac2fa2e414ba752645816167f943e
13772:X 27 Jan 01:05:25.514 # +monitor master mymaster 127.0.0.1 6379 quorum 2

````
启动Sentinel的时候,它要执行以下操作

1. 初始化服务器。
2. 将普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。
3. 初始化 Sentinel 状态。
4. 根据给定的配置文件， 初始化 Sentinel 的监视主服务器列表。
5. 创建连向主服务器的网络连接。


## 初始化服务器

首先， 因为 Sentinel 本质上只是一个运行在特殊模式下的 Redis 服务器， 所以启动 Sentinel 的第一步， 就是初始化一个普通的 Redis 服务器， 具体的步骤和《服务器》一章介绍的类似。

不过， 因为 Sentinel 执行的工作和普通 Redis 服务器执行的工作不同， 所以 Sentinel 的初始化过程和普通 Redis 服务器的初始化过程并不完全相同。

## 使用 Sentinel 专用代码

比如说， 普通 Redis 服务器使用 redis.h/REDIS_SERVERPORT 常量的值作为服务器端口：
````
#define REDIS_SERVERPORT 6379
````
而 Sentinel 则使用 sentinel.c/REDIS_SENTINEL_PORT 常量的值作为服务器端口：
````
#define REDIS_SENTINEL_PORT 26379
````

 Redis 服务器不能执行诸如 SET 、 DBSIZE 、 EVAL 等等这些命令 —— 因为服务器根本没有在命令表中载入这些命令： PING 、 SENTINEL 、 INFO 、 SUBSCRIBE 、 UNSUBSCRIBE 、 PSUBSCRIBE 和 PUNSUBSCRIBE 这七个命令就是客户端可以对 Sentinel 执行的全部命令了。
 

## 初始化Sentinel状态

在应用了 Sentinel 的专用代码之后， 接下来， 服务器会初始化一个 sentinel.c/sentinelState 结构

````
struct sentinelState {

    // 当前纪元，用于实现故障转移
    uint64_t current_epoch;

    // 保存了所有被这个 sentinel 监视的主服务器
    // 字典的键是主服务器的名字
    // 字典的值则是一个指向 sentinelRedisInstance 结构的指针
    dict *masters;

    // 是否进入了 TILT 模式？
    int tilt;

    // 目前正在执行的脚本的数量
    int running_scripts;

    // 进入 TILT 模式的时间
    mstime_t tilt_start_time;

    // 最后一次执行时间处理器的时间
    mstime_t previous_time;

    // 一个 FIFO 队列，包含了所有需要执行的用户脚本
    list *scripts_queue;

} sentinel;

````

## 初始化 Sentinel 状态的 masters 属性

Sentinel 状态中的 masters 字典记录了所有被 Sentinel 监视的主服务器的相关信息， 其中：

1. 字典的键是被监视主服务器的名字。
2. 而字典的值则是被监视主服务器对应的 sentinel.c/sentinelRedisInstance 结构。

````
typedef struct sentinelRedisInstance {

    // 标识值，记录了实例的类型，以及该实例的当前状态
    int flags;

    // 实例的名字
    // 主服务器的名字由用户在配置文件中设置
    // 从服务器以及 Sentinel 的名字由 Sentinel 自动设置
    // 格式为 ip:port ，例如 "127.0.0.1:26379"
    char *name;

    // 实例的运行 ID
    char *runid;

    // 配置纪元，用于实现故障转移
    uint64_t config_epoch;

    // 实例的地址
    sentinelAddr *addr;

    // SENTINEL down-after-milliseconds 选项设定的值
    // 实例无响应多少毫秒之后才会被判断为主观下线（subjectively down）
    mstime_t down_after_period;

    // SENTINEL monitor <master-name> <IP> <port> <quorum> 选项中的 quorum 参数
    // 判断这个实例为客观下线（objectively down）所需的支持投票数量
    int quorum;

    // SENTINEL parallel-syncs <master-name> <number> 选项的值
    // 在执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量
    int parallel_syncs;

    // SENTINEL failover-timeout <master-name> <ms> 选项的值
    // 刷新故障迁移状态的最大时限
    mstime_t failover_timeout;

    // ...

} sentinelRedisInstance;

````

Sentinel启动时会载入以下内容的配置文件

````
#####################
# master1 configure #
#####################

sentinel monitor master1 127.0.0.1 6379 2

sentinel down-after-milliseconds master1 30000

sentinel parallel-syncs master1 1

sentinel failover-timeout master1 900000

#####################
# master2 configure #
#####################

sentinel monitor master2 127.0.0.1 12345 5

sentinel down-after-milliseconds master2 50000

sentinel parallel-syncs master2 5

sentinel failover-timeout master2 450000

````

![master1的实例结构](/images/redis/sentinel/master1的实例结构.png)

![master2的实例结构](/images/redis/sentinel/master2的实例结构.png)

![sentinel状态以及master字典](/images/redis/sentinel/sentinel状态以及master字典.png)

## 创建连向主服务器的网络连接

对于每个被 Sentinel 监视的主服务器来说， Sentinel 会创建两个连向主服务器的异步网络连接：

1. 一个是命令连接， 这个连接专门用于向主服务器发送命令， 并接收命令回复。
2. 另一个是订阅连接， 这个连接专门用于订阅主服务器的 __sentinel__:hello 频道。


# 获取主服务器的信息

Sentinel默认会以每十秒一次的频率,通过命令连接向被监视的**主服务**器发送INFO命令.并通过分析info命令的回复来获取主服务器的当前信息.

![info命令的回复](/images/redis/sentinel/info命令的回复.png)

Sentinel在分析info命令返回的信息中对应的从服务器的实例是否在slave字典中存在而做出相应的操作.

![主服务器和它的三个从服务器](/images/redis/sentinel/主服务器和它的三个从服务器.png)

# 获取从服务器信息

当Sentinel发现主服务器有新的从服务器出现时.Sentinel除了会为这个新的从服务器创建相应的实例结构外,Sentinel还会创建连接到从服务器的命令连接和订阅连接.

![sentinel与各个从服务器建立命令连接和订阅连接](/images/redis/sentinel/sentinel与各个从服务器建立命令连接和订阅连接.png)

Sentinel默认会以每十秒一次的频率,通过命令连接向被监视的**从服务**器发送INFO命令.并通过分析info命令的回复来获取主服务器的当前信息.

![从服务器的回复](/images/redis/sentinel/从服务器的回复.png)

![从服务器的实例结构](/images/redis/sentinel/从服务器的实例结构.png)

# 向主服务器和从服务器发送信息

默认情况下,Sentinel会以每两秒一次的频率,通过命令连接向所有被监视的主服务器和从服务器发送以下格式命令:

![sentinel发送命令的格式](/images/redis/sentinel/sentinel发送命令的格式.png)

# 接收来自主服务器和从服务器的频道信息

每个Sentinel连接的服务器.Sentinel既会通过命令连接向服务器的_Sentinel_:hello频道发送信息.又通过订阅连接从服务器的_Sentinel_:hello频道接收信息.

对于多个Sentinel来说,一个Sentinel发送的信息会被其它Sentinel接收到,以更新他们对监视服务器的认知.

## 更新Sentinels字典

每个Sentinel实例中的Sentinels字典中保存这其它Sentinel实例的信息.各个Sentinel的ip和port就是该Sentinel字典中的key,而value就是Sentinel的实体结构

每次接收来自其它实例的信息时,各个实例都会更新其它实例在自己字典中的信息.

## 创建连向其它Sentinel的命令连接

当Sentinel通过频道信息发现一个新的Sentinel时,它不仅会为Sentinel在Sentinels字典中创建相应的实例结构,还会创建一个连向新Sentinel的命令连接.而新的Sentinel也会创建连向这个Sentinel的连接,最终监视同一个服务器的多个Sentinel会形成相互连接的网络

这样,各个Sentinel会通过其它Sentinel发来的命令交换信息.

- 注意 Sentinel之间不会创建订阅连接.

# 检测主观下线状态

默认Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例(主从服务器,其它Sentinel在内)发送ping命令,通过他们的回复来判断他们是否在线.

Sentinel配置文件中的Down-after-milliseconds选项制定了Sentinel判断实例进入主观下线所需的时间长度.如果在该时间内无回复,则Sentinel会改变这个实例的对应的实例结构,在flags属性中打开SRI_S_Down表示,标志这该实例进入主管下线状态.

由于多个Sentinel的毫秒数设置的可能不同,所以各个Sentinel主观认为某个实例是否下线时不同的.

# 检查客观下线状态

当Sentinel将一个主服务器判断为主管下线之后,为了确认这个主服务器是否真的下线了,他会向同样监视这个主服务器的Sentinel进行询问.当从其它的Sentinel那接收到足够数量的已下线判断后,Sentinel会将主服务器判定为客观下线,并对主服务器执行故障转移操作

## 发送Sentinel is-master-down-by-addr命令

![Sentinel is-master-down-by-addr各个参数的意义](/images/redis/sentinel/Sentinel is-master-down-by-addr各个参数的意义.png)

## 接收Sentinel is-master-down-by-addr命令

![Sentinel is-master-down-by-addr回复的意义](/images/redis/sentinel/Sentinel is-master-down-by-addr回复的意义.png)

## 接收Sentinel is-master-down-by-addr的回复

根据Sentinel同意主服务器已下线的数量(quorum参数的值),当达到所需的数量,则将主服务器的flags属性的SRI_O_DOWN标示打开,表示主服务器已经进入客观下线状态

Sentinel monitor master 127.0.0.1 6379 2 包括当前的Sentinel在日,有2个Sentinel觉得主服务器已经下线,则当前Sentinel才会将主服务器判断为客观下线.

# 选举领头 Sentinel

当一个主服务器被判定为客观下线时.监视这个下线主服务器的各个Sentinel会进行协商,选举一个领头Sentinel,并由它对下线主服务器进行故障转移.

选举领头Sentinel的法则:

![选举领头Sentinel的法则1](/images/redis/sentinel/选举领头Sentinel的法则1.png)

![选举领头Sentinel的法则2](/images/redis/sentinel/选举领头Sentinel的法则2.png)

# 故障转移

领头的Sentinel对已下线的主服务器执行故障转移,操作如下

1. 在已下线的主服务器属下的所有从服务器里面,选一个从服务器将其转换为主服务器.
2. 让已下线的服务器属下的所有从服务器改为复制新的主服务器.
3. 将已下线的主服务器设置为新的主服务器的从服务器,待其上线后,它将成为新的额主服务器的从服务器.

## 选出的新的主服务器

故障转移操作的第一步就是在已下线的主服务器的属下的所有从服务器中,选出一个状态良好.数据完整的从服务器,然后向这个从服务器发送slave no one命令.将它由从服务器变为主服务器.

主服务的挑选规则:

![主服务的挑选规则](/images/redis/sentinel/主服务的挑选规则.png)

## 修改从服务器的复制目标

当新的主服务器出现之后,领头Sentinel下一步做的就是让所有的从服务器去复制新的主服务器.这一动作通过**领头Sentinel**向从服务器发送SLAVEOF命令来实现

![让从服务器复制新的主服务器](/images/redis/sentinel/让从服务器复制新的主服务器.png)

![server3和server4成为server2的从服务器](/images/redis/sentinel/server3和server4成为server2的从服务器.png)

## 让旧的主服务器成为从服务器

由于旧的主服务器已经下线,所以这种操作时保存在在server1对应的实例结构里面的.当server1重新上线后,Sentinel会向它发送SLAVEOF命令,让server1成为server2的从服务器.

![server1重新上线成为server2的从服务器](/images/redis/sentinel/server1重新上线成为server2的从服务器.png)









 




































