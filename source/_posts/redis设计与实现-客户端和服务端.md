---
title: redis设计与实现-客户端和服务端
date: 2018-01-16 01:43:55
tags: redis客户端,redis服务端
categories: redis
copyright: true

---

# 客户端

redis服务器与客户端是一对多的关系,使用由I/O多路复用技术实现的文件事件处理器,redis服务器单线程和单进程的方式来处理命令请求,并与多个客户端进行网络通信.

redisClient结构保存了客户端当前的状态信息:

1. 客户端的套接字描述
2. 客户端的名字
3. 客户端的标志值 flag
4. 执行客户端正在使用的数据库的指针,以及数据库的号码
5. 客户端当前要执行的命令/命令的参数/命令参数的个数/指向命令实现函数的指针
6. 客户端输入和输出缓冲区
7. 客户端复制状态信息以及进行复制所需要的数据结构
8. 客户端执行BRPOP BLPOP等等列表阻塞命令时使用的数据结构
9. 客户端的事务状态以及watch命令用到的数据结构
10. 身份验证标志
11. 客户端的创建时间,客户端和服务端最后的通信时间等

Redis服务器状态结构的**clients属性是一个链表**,这个链表保存了所有连接的客户端的状态结构

![redisServer的clients](/images/redis/serverClient/redisServer的clients.png)

![clients链表](/images/redis/serverClient/clients链表.png)

## 客户端属性

一种是通用的属性,另一种是特定功能的属性

### 套接字描述

td属性 fd属性的值可以是-1或者大于-1的整数:

伪客户端的fd属性的值为-1,它处理的命令来自AOF文件或者Lua脚本,而不是网络.

普通客户端的fd属性值为大于-1的整数,它使用套接字来与服务器进行通讯,所以服务器使用fd属性来记录客户端套接字的描述符.

客户端的描述信息

````
27.0.0.1:6379> CLIENT LIST
id=2 addr=127.0.0.1:53958 fd=5 name= age=25 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client


````

### 名字

默认的客户端时没有名字的,可以使用client setname来设置名字

````
127.0.0.1:6379> CLIENT SETNAME testName
OK
127.0.0.1:6379> CLIENT GETNAME
"testName"
127.0.0.1:6379> CLIENT LIST
id=2 addr=127.0.0.1:53958 fd=5 name=testName age=198 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client

````

### 标志

flags记录客户端的角色,可以是一个或者多个

flags=<flag1>|<flag2> 

![client的flags1](/images/redis/serverClient/client的flags1.png)

![client的flags2](/images/redis/serverClient/client的flags2.png)

### 输入缓冲区

在客户端输入以下命令:

````
set key value

````
以下展示SDS的值以及querybuf属性的样子

![querybuf属性的样子](/images/redis/serverClient/querybuf属性的样子.png)

### 命令与命令参数

服务器将客户端端发送的请求保存到客户端状态的querybuf属性之后,服务器将对命令请求的状态进行分析,并将得出的命令和参数的格式分别保存到客户端状态的argv属性和argc属性

argv属性是一个数组,数组中的每个项都是一个字符串对象,其中arg[0]时要执行的命令,而后的其它项都是传给命令的参数

argc属性则负责记录argv数组的长度

![argv和argc属性的示例](/images/redis/serverClient/argv和argc属性的示例.png)

### 命令的实现函数

服务器会根据argv[0]去命令表中查找命令所对应的命令实现函数.

命令表如下所示,它时一个字典,字典的键时一个SDS结构,保存了命令的名字,字典的值是命令所对应的redisCommand结构,这个结构保存了命令的实现函数.命令的标志.命令应该给定的参数个数.命令的执行次数以及总消耗时长等信息

![查找命令并设置cmd属性](/images/redis/serverClient/查找命令并设置cmd属性.png)

当程序在命令表中成功的找到argv[0]结构时,它会将客户端状态的cmd指针指向redisClient结构,之后服务器就使用cmd属性指向的redisCommand结构,以及其它一些参数来执行制定的命令.

### 输出缓冲区

执行命令所得的命令的回复会被保存到客户端状态的输出缓冲区中,每个客户端都有2个输出缓冲区

#### 固定大小

固定大小的缓冲区用于保存那些长度比较小的回复,ok,简短的字符串值等

![固定大小的缓冲区](/images/redis/serverClient/固定大小的缓冲区.png)


#### 可变大小

可变大小的缓冲区用于保存那些长度比较大的回复

可变大小缓冲区由reply链表和一个或者多个字符串对象组成

![可变大小缓冲区示例](/images/redis/serverClient/可变大小缓冲区示例.png)

### 身份验证

客户端状态的authenticated属性用于记录客户端是否通过了身份验证: 值为0表示未通过身份验证,1表示已经通过身份验证,使用auth命令进行验证

### 时间

ctime属性记录了创建客户端的时间,client list命令的age域记录了这个秒数

lastinteraction记录了与客户端最后互动的时间,client list命令的idle记录了空闲了多久

## 客户端的创建和关闭

### 创建客户端

如果客户端是通过网络连接与服务器进行连接的普通客户端,那么connect函数连接到服务器时,服务器就会调用连接事件处理器为客户端创建相应的客户端状态,并将这个新的客户端状态添加到服务器链表的末尾

![服务器状态结构的clients链表](/images/redis/serverClient/服务器状态结构的clients链表.png)

### 关闭普通客户端

关闭的原因如下:

![客户端关闭的原因如下](/images/redis/serverClient/客户端关闭的原因如下.png)

### Lua脚本的伪客户端

服务器会在初始化时创建负责执行Lua脚本中包含的Redis命令的伪客户端.伪客户端在服务器运行的整个生命周期都会一直存在,只有服务器被关闭时,这个客户端才会被关闭

### AOF伪客户端

载入AOF文件时,会创建用于执行AOF文件中包含的redis命令的伪客户端,载入完成则关闭这个伪客户端.

# 服务器

redis服务器负责与多个客户端建立网络连接,处理客户端发送的命令请求,在数据库中保存客户端执行命令所产生的数据,并通过资源管理来维持服务器自身的运转.

## 命令请求的执行过程

````
set key value 

````

1. 客户端向服务器发送命令请求 SET KEY VALUE 。
2. 服务器接收并处理客户端发来的命令请求 SET KEY VALUE ， 在数据库中进行设置操作， 并产生命令回复 OK 。
3. 服务器将命令回复 OK 发送给客户端。
4. 客户端接收服务器返回的命令回复 OK ， 并将这个回复打印给用户观看。

### 发送命令请求

Redis 服务器的命令请求来自 Redis 客户端， 当用户在客户端中键入一个命令请求时， 客户端会将这个命令请求转换成协议格式， 然后通过连接到服务器的套接字， 将协议格式的命令请求发送给服务器

![客户端接收并发送请求命令的过程](/images/redis/serverClient/客户端接收并发送请求命令的过程.png)

### 读取命令请求

服务器读取命令的过程

1. 读取套接字中协议格式的命令请求， 并将其保存到客户端状态的输入缓冲区里面。
2. 对输入缓冲区中的命令请求进行分析， 提取出命令请求中包含的命令参数， 以及命令参数的个数， 然后分别将参数和参数个数保存到客户端状态的 argv 属性和 argc 属性里面。
3. 调用命令执行器， 执行客户端指定的命令。

![客户端状态中的命令请求](/images/redis/serverClient/客户端状态中的命令请求.png)

![客户端状态中的argv和argc参数](/images/redis/serverClient/客户端状态中的argv和argc参数.png)


## 命令执行器

命令执行器要做的第一件事就是根据客户端状态的 argv[0] 参数， 在命令表（command table）中查找参数所指定的命令， 并将找到的命令保存到客户端状态的 cmd 属性里面。

### 查找命令的实现

redisCommand 结构的主要属性

![redisCommand结构的主要属性](/images/redis/serverClient/redisCommand结构的主要属性.png)

sflags属性的标识

![sflags属性的标识](/images/redis/serverClient/sflags属性的标识.png)


SET 命令的名字为 "set" ， 实现函数为 setCommand ； 命令的参数个数为 -3 ， 表示命令接受三个或以上数量的参数； 命令的标识为 "wm" ， 表示 SET 命令是一个写入命令， 并且在执行这个命令之前， 服务器应该对占用内存状况进行检查， 因为这个命令可能会占用大量内存。

GET 命令的名字为 "get" ， 实现函数为 getCommand 函数； 命令的参数个数为 2 ， 表示命令只接受两个参数； 命令的标识为 "r" ， 表示这是一个只读命令。

![命令表](/images/redis/serverClient/命令表.png)

继续之前 SET 命令的例子， 当程序以图 14-3 中的 argv[0] 作为输入， 在命令表中进行查找时， 命令表将返回 "set" 键所对应的 redisCommand 结构， 客户端状态的 cmd 指针会指向这个 redisCommand 结构， 如图 14-5 所示。

![设置客户端状态的cmd指针](/images/redis/serverClient/设置客户端状态的cmd指针.png)

### 进行预备操作

在以上操作之后,在真正执行命令之前还要进行一些预备操作

1. 检查客户端状态的 cmd 指针是否指向 NULL ， 如果是的话， 那么说明用户输入的命令名字找不到相应的命令实现， 服务器不再执行后续步骤， 并向客户端返回一个错误。
2. 根据客户端 cmd 属性指向的 redisCommand 结构的 arity 属性， 检查命令请求所给定的参数个数是否正确， 当参数个数不正确时， 不再执行后续步骤， 直接向客户端返回一个错误。 比如说， 如果 redisCommand 结构的 arity 属性的值为 -3 ， 那么用户输入的命令参数个数必须大于等于 3 个才行。
3. 检查客户端是否已经通过了身份验证， 未通过身份验证的客户端只能执行 AUTH 命令， 如果未通过身份验证的客户端试图执行除 AUTH 命令之外的其他命令， 那么服务器将向客户端返回一个错误。
4. 如果服务器打开了 maxmemory 功能， 那么在执行命令之前， 先检查服务器的内存占用情况， 并在有需要时进行内存回收， 从而使得接下来的命令可以顺利执行。 如果内存回收失败， 那么不再执行后续步骤， 向客户端返回一个错误。
5. 如果服务器上一次执行 BGSAVE 命令时出错， 并且服务器打开了 stop-writes-on-bgsave-error 功能， 而且服务器即将要执行的命令是一个写命令， 那么服务器将拒绝执行这个命令， 并向客户端返回一个错误。
6. 如果客户端当前正在用 SUBSCRIBE 命令订阅频道， 或者正在用 PSUBSCRIBE 命令订阅模式， 那么服务器只会执行客户端发来的 SUBSCRIBE 、 PSUBSCRIBE 、 UNSUBSCRIBE 、 PUNSUBSCRIBE 四个命令， 其他别的命令都会被服务器拒绝。
7. 如果服务器正在进行数据载入， 那么客户端发送的命令必须带有 l 标识（比如 INFO 、 SHUTDOWN 、 PUBLISH ，等等）才会被服务器执行， 其他别的命令都会被服务器拒绝。
8. 如果服务器因为执行 Lua 脚本而超时并进入阻塞状态， 那么服务器只会执行客户端发来的 SHUTDOWN nosave 命令和 SCRIPT KILL 命令， 其他别的命令都会被服务器拒绝。
9. 如果客户端正在执行事务， 那么服务器只会执行客户端发来的 EXEC 、 DISCARD 、 MULTI 、 WATCH 四个命令， 其他命令都会被放进事务队列中。
10. 如果服务器打开了监视器功能， 那么服务器会将要执行的命令和参数等信息发送给监视器。

### 调用命令的实现函数

````
// client 是指向客户端状态的指针

client->cmd->proc(client);

````


以上代码相当于执行setCommand(client)

![客户端状态](/images/redis/serverClient/客户端状态.png)

被调用的命令实现函数会执行指定的操作， 并产生相应的命令回复， 这些回复会被保存在客户端状态的输出缓冲区里面（buf 属性和 reply 属性）， 之后实现函数还会为客户端的套接字关联命令回复处理器， 这个处理器负责将命令回复返回给客户端。

### 执行后续操作

1. 如果服务器开启了慢查询日志功能， 那么慢查询日志模块会检查是否需要为刚刚执行完的命令请求添加一条新的慢查询日志。
2. 根据刚刚执行命令所耗费的时长， 更新被执行命令的 redisCommand 结构的 milliseconds 属性， 并将命令的 redisCommand 结构的 calls 计数器的值增一。
3. 如果服务器开启了 AOF 持久化功能， 那么 AOF 持久化模块会将刚刚执行的命令请求写入到 AOF 缓冲区里面。
4. 如果有其他从服务器正在复制当前这个服务器， 那么服务器会将刚刚执行的命令传播给所有从服务器。

### 将命令回复给客户端

服务器命令实现函数为套接字关联命令回复处理器,将客户端buf属性中回复信息以协议格式的命令回复给客户端.

### 客户端接收并打印命令回复

![客户端接收并打印命令回复](/images/redis/serverClient/客户端接收并打印命令回复.png)

## serverCron 函数

serverCron函数默认每100毫秒执行一次,这个函数负责管理服务器资源

### 更新服务器的时间缓存

服务器中有好多命令需要获取当前时间,为了减少系统调用的执行次数.服务器状态中的unixtime和mstime属性被作为当前时间的缓存

![redisServer中的时间属性](/images/redis/serverClient/redisServer中的时间属性.png)

serverCron函数默认会100毫秒一次的频率更新unixtime属性和mstime属性.所以这两个属性记录的时间精度不高,

对于打印日志,更新服务器的LRU时钟,决定是否执行持久化任务,计算服务器上线时间这类对时间精度不高的功能上使用以上时间属性.对键设置过期时间 添加慢查询日志这种需要搞定的时间功能来说还是需要获取最准确的系统当前时间

### 更新LRU时钟

服务器状态中的lruclock属性保存了服务器的LRU时钟,这个服务器时间缓存的一种,默认每10秒更新一次时钟缓存.用于计算键的idle

每个redis对象都有一个lru属性,保存了对象最后一次被访问的时间

当服务器计算一个数据库键的空转时间,可以使用服务器的lruclock减去键的lru属性

因为serverCron函数会以每10秒一次的频率更新lruClock的值,所以计算的idle不是非常精确的

### 更新服务器美妙执行命令的次数

serverCron函数中的trackOperationPerSecond函数会以每100毫秒一次的频率执行,这个函数以抽样的方式计算服务器在最近一秒钟处理的命令请求数量

````
127.0.0.1:6379> info stats
# Stats
total_connections_received:1
total_commands_processed:5
instantaneous_ops_per_sec:0
total_net_input_bytes:145
total_net_output_bytes:11352
instantaneous_input_kbps:0.00
instantaneous_output_kbps:0.00
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
migrate_cached_sockets:0

````

### 更新服务器内存峰值记录

used_memory_peak和used_memory_peak_human参数分别以两种形式记录服务器内存峰值

````
127.0.0.1:6379> info memory
# Memory
used_memory:821744
used_memory_human:802.48K
used_memory_rss:9351168
used_memory_rss_human:8.92M
used_memory_peak:821744
used_memory_peak_human:802.48K
total_system_memory:8053633024
total_system_memory_human:7.50G
used_memory_lua:37888
used_memory_lua_human:37.00K
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
mem_fragmentation_ratio:11.38
mem_allocator:jemalloc-4.0.3

````

### 处理sigterm信号

在启动服务器时,redis会为服务器进程的sigterm信号关联处理器sigtermHandler函数,这个信号处理器负责在服务器接收到sigterm信号时打开关标识

![sigtermHandler函数](/images/redis/serverClient/sigtermHandler函数.png)

服务器一接收到sigterm信号就会关闭服务器,服务器要拦截sigterm信号,在关闭服务器之前进行RDB操作

### 管理客户端资源

serverCron函数每次执行都会调用clientsCron函数,该clientsCron函数会对客户端进行检查

1. 如果客户端与服务器连接已经超时(客户端与服务器很长时间都没有互动),那么程序释放这个客户端

2. 如果客户端在上一次执行命令之后,输入缓冲区的大小超过了一定的长度,那么程序会释放客户端的当前的缓冲区,并重新创建一个默认的缓冲区

### 管理数据库资源

serverCron函数每次执行都会调用databaseCron函数,该databaseCron函数会对数据库进行检查,删除其中的过期键.

### 执行被延迟的BGREWRITEAOF

在服务器执行BGSAVE命令的期间,如果客户端向服务器发来BGREWRITEAOF命令,那么服务器会将BGREWRITEAOF延迟到BGSAVE命令之后执行 

redisServer的aof_rewrite_scheduled标识记录了服务器是否延迟了BGREWRITEAOF命令,如果被延迟,则将该标识置为1

每次调用serverCron函数执行,函数都检查bgsave和BGREWRITEAOF命令是否在执行,如果都没有执行,并且aof_rewrite_scheduled属性的值为1,那么服务器就会执行之前被推延的BGREWRITEAOF命令

### 检查持久化操作的运行状态

服务器使用rdb_child_pid属性和aof_child_pid属性记录执行bgsave命令和BGREWRITEAOF命令的子进程的ID,这两个属性也可以检查bgsave命令或者BGREWRITEAOF是否正在执行

![持久化操作的运行状态的参数](/images/redis/serverClient/持久化操作的运行状态的参数.png)

- 每次serverCron函数执行都会检查这两个pid属性的值是否为-1,

      只有其中一个不为-1,程序就会执行wait3函数,检查子进程是否有信号发来服务器进程 如果有信号到达,表示新的RDB文件生成完成或者新的AOF文件重写完毕.服务器需要进行后续操作,比如新的RDB文件替换现有的RDB文件,新的AOF文件替换现有的AOF文件

      如果2个进程id都不是-1则要进行以下检查:检查是否有BGREWRITEAOF命令被延迟;是否满足自动保存条件(bgsave执行的条件);AOF重写的条件是否满足

### AOF缓冲区的内容写到AOF文件

如果服务器开启了AOF功能,并且AOF文件中还有待写入的数据,那么serverCron函数会调用相应的函数将数据写入到AOF文件

### 关闭异步客户端

关闭那些输入缓冲区超出大小限制的客户端

### 增加cronloops计数器的值

服务器状态的cronloops记录了serverCron函数执行的次数(唯一的功能是,实现每执行N次就执行一次制定的代码)

## 初始化服务器

### 初始化服务器状态结构

即将redisServer类型的实例变量server作为服务器的状态,并为结构中的各个属性设置默认值,调用initServerConfig函数做以下工作,为服务器赋默认值

1. 设计服务器的运行id
2. 设计服务器的默认运行频率
3. 设置服务器的默认配置文件路径
4. 设置服务器的运行架构
5. 设置服务器的默认端口号
6. 设置服务器的默认RDB持久化条件和AOF持久化条件
7. 初始化服务器的LRU时钟
8. 创建命令表

### 载入配置文件

用户可以制定参数或者制定配置文件,对服务器的默认配置进行修改,是对初始化服务器结构的参数进行修改.这是第二步操作.

直接指定参数

````
zhuningning@ubuntu:/usr/local/redis-3.2.10/src$ ./redis-server -- port 10086 &
[1] 11639
zhuningning@ubuntu:/usr/local/redis-3.2.10/src$ 11639:M 18 Jan 02:09:01.264 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.2.10 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 10086
 |    `-._   `._    /     _.-'    |     PID: 11639
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

11639:M 18 Jan 02:09:01.266 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11639:M 18 Jan 02:09:01.266 # Server started, Redis version 3.2.10
11639:M 18 Jan 02:09:01.267 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
11639:M 18 Jan 02:09:01.267 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
11639:M 18 Jan 02:09:01.267 * DB loaded from disk: 0.000 seconds
11639:M 18 Jan 02:09:01.267 * The server is now ready to accept connections on port 10086

````

或者指定配置文件

````
zhuningning@ubuntu:/usr/local/redis-3.2.10/src$ ./redis-server ../redis.conf &
[1] 11765
zhuningning@ubuntu:/usr/local/redis-3.2.10/src$ 11765:M 18 Jan 02:10:21.493 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.2.10 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 11765
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

11765:M 18 Jan 02:10:21.495 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11765:M 18 Jan 02:10:21.495 # Server started, Redis version 3.2.10
11765:M 18 Jan 02:10:21.495 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
11765:M 18 Jan 02:10:21.495 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
11765:M 18 Jan 02:10:21.495 * DB loaded from disk: 0.000 seconds
11765:M 18 Jan 02:10:21.495 * The server is now ready to accept connections on port 6379

````

### 初始化服务器数据结构

第三步 就是调用initServer初始化数据结构,包括:

server.clients链表,这个链表包含了所有与服务器连接的客户端redisClient,结构实例

server.db数组,包含了服务器的所有数据库

server.pubsub_channelszidian关于频道订阅信息的

用于执行lua脚本的环境server.lua

用于保存慢查询日志的server.slowlog属性

为以上的数据机构分配内存,除此之外还包括以下:

![initServer函数的其它操作](/images/redis/serverClient/initServer函数的其它操作.png)

### 还原数据库状态

如果开启了AOF持久化功能,则使用AOF文件还原数据库,否则使用RDB文件还原数据库

````
11765:M 18 Jan 02:10:21.495 * DB loaded from disk: 0.000 seconds
````

### 执行时间循环

初始化的最后异步打印

````
11765:M 18 Jan 02:10:21.495 * The server is now ready to accept connections on port 6379
````

之后开始执行时间循环.

这样服务器这样就可以接收客户端的连接请求,以及接收客户端的命令了.





