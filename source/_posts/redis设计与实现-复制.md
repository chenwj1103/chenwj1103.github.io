---
title: redis设计与实现-复制
copyright: true
date: 2018-01-24 00:43:00
tags: redis复制
categories: redis

---

redis中可以通过slaveof命令或者配置slaveof选项,让一个服务器去复制(replicate)另一个服务器,被复制的服务器为master,对master进行复制的为slave.

从服务器和主服务保存的数据时相同的.这个称为数据库状态一致.

# 旧版复制功能的实现

复制功能分为同步(sync)和命令传播(command propagate) 两个操作

1. 同步是从服务器的数据库状态更新至主服务器当前所处的数据库状态.
2. 命令传播操作是当主服务器的数据库状态被修改,导致主从服务器的数据库状态出现不一致的时候,让主从服务器的数据库状态回到一致状态.

## 同步

从服务器对主服务器的同步操作需要通过向主服务器发送 SYNC 命令来完成， 以下是 SYNC 命令的执行步骤：

1. 从服务器向主服务器发送 SYNC 命令。
2. 收到 SYNC 命令的主服务器执行 BGSAVE 命令， 在后台生成一个 RDB 文件， 并使用一个缓冲区记录从现在开始执行的所有写命令。
3. 当主服务器的 BGSAVE 命令执行完毕时， 主服务器会将 BGSAVE 命令生成的 RDB 文件发送给从服务器， 从服务器接收并载入这个 RDB 文件， 将自己的数据库状态更新至主服务器执行 BGSAVE 命令时的数据库状态。
4. 主服务器将记录在缓冲区里面的所有写命令发送给从服务器， 从服务器执行这些写命令， 将自己的数据库状态更新至主服务器数据库当前所处的状态。

![主从服务器同步的例子](/images/redis/replicate/主从服务器同步的例子.png)

## 命令传播

当同步完成之后,主从服务器的状态在当前时刻时一致的.当主库发生写操作,此时主从的数据时不一致的.主库需要将客户端传来的写操作的命令传递给从库.这样主从数据库才可以保持一致.

# 旧版复制功能的缺陷

主从复制分为以下两种情况:

1. 初次复制,从服务器从来没有复制过任何主服务器,或者从服务器要复制的主服务器和上一次复制的主服务器不同
2. 断线后的复制,处于命令传播阶段的主从服务器因为网络原因中断了复制,网络恢复后从服务器对主服务器的复制.(断线重连后从服务器需要向主服务器发送sync命令)

![断线后重新复制主服务器的例子](/images/redis/replicate/断线后重新复制主服务器的例子.png)

其实断线这段时间,主服务器的写操作产生的数据是主从服务器之间的不同.但旧版的复制功能却采用sync命令,这样效率其实非常低.

# 新版复制功能的实现

为了解决旧版掉线重连的同步数据地下的缺陷,有了新版复制功能的实现.

新版的复制功能采用PSYNC代替SYNC命令.PSYNC具有完整重同步和部分重同步2种操作,其中完整重同步和SYNC的基本一致,部分重同步是在网络恢复后,将断线期间的写命令发送给从服务器.

![PSYNC命令进行断线后重复制](/images/redis/replicate/PSYNC命令进行断线后重复制.png)

# 部分重同步的实现

主要包括以下三小结

## 复制偏移量

主从服务器都会维护一个复制偏移量.主服务器每次向从服务器传播N个字节,就会将自己的偏移量加上N,从服务器收到偏移量N就会为自己的偏移量加上N.

![拥有相同偏移量的一主三从服务器](/images/redis/replicate/拥有相同偏移量的一主三从服务器.png)

## 复制积压缓冲区

复制积压缓冲区是由主服务器维护的一个固定长度先进先出的队列.默认大小为1M.

当主服务器进行命令传播时,不仅将命令发送给所有的从服务器,还会将命令写入到复制积压缓冲区里面.

![复制积压缓冲区的构造](/images/redis/replicate/复制积压缓冲区的构造.png)

当从服务器重新连接上主服务器后,从服务器会通过PSYNC命令将自己的复制偏移量offset发送给服务器,主服务会根据这个offset来决定执行什么操作.

如果offset偏移量之后的数据(offset+1后的数据),仍然存在与积压缓冲区中,则主服务会对从服务器执行部分重同步操作,否则自己完整重同步操作.

## 服务器运行id

除了复制偏移量和复制积压缓冲区外,实现部分重同步还需要用到服务器的运行ID.

````
127.0.0.1:6379> info server
# Server
redis_version:3.2.10
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:e84c37c0154af52a
redis_mode:standalone
os:Linux 4.4.0-109-generic x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:5.4.0
process_id:22916
run_id:1976d153c2f5dd6d9c62808b92dddbe572950c88 //40个随机的16进制字符组成
tcp_port:6379
uptime_in_seconds:180
uptime_in_days:0
hz:10
lru_clock:6863212
executable:/usr/local/redis-3.2.10/src/./redis-server
config_file:/usr/local/redis-3.2.10/redis.conf

````

从服务器对主服务器进行初次同步的时候,主服务会将自己的run id传递给从服务器.断线重连后会将此id传递给主服务器,如果id和主服务器的相同,则尝试执行部分同步操作.否则执行全同步操作.


# PSYNC命令的实现

PSYNC命令执行完整重同步和部分重同步时可能遇到的情况.

![PSYNC命令执行完整重同步和部分重同步时可能遇到的情况](/images/redis/replicate/PSYNC命令执行完整重同步和部分重同步时可能遇到的情况.png)

# 复制的实现

从服务通过 SLAVEOF <master_ip> <master_port> 复制主服务的数据

## 设置主服务器的地址和端口

执行SLAVEOF 127.0.0.1 6379命令后,从服务将主服务器的ip和端口设置到自己的服务器属性里.

![从服务器的服务器状态](/images/redis/replicate/从服务器的服务器状态.png)

## 建立套接字连接

从服务器设置ip和端口号成功后,创建连向主服务器的套接字连接.创建成功后,从服务器会为这个套接字关联一个专门用于处理复制工作的文件事件处理器.

主服务器在接收从服务器的套接字连接之后,将为该套接字创建相应的客户端状态.而此时从服务器可以看做时连接向主服务器的客户端.此时从服务器同事具有服务器和客户端两个身份.

## 发送ping命令

虽然建立起连接,但是主从服务器从未通信,

1. 以此来检测套接字的读写状态是否正常.
2. 检测主服务器能否正常处理命令.

收到的回复

1. timeout表示连接状态不佳.
2. 返回一个错误,表示暂时没法处理命令.
3. pong,正常.

## 身份验证

返回pong之后,如果从服务器设置了masterauth参数,那么进行身份验证.否者不进行身份验证.

![从服务进行身份验证可能遇到的问题](/images/redis/replicate/从服务进行身份验证可能遇到的问题.png)

## 发送端口信息

在身份验证步骤之后,从服务器将执行命令REPLCONF listening-port<port-num>,向主服务器发送从服务器的监听端口号

主服务在接收到这个命令之后,会将端口号记录在从服务器对应的客户端状态的slave_listening_port属性中.目前唯一的作用就是info repication会打印监听的端口号

## 同步

之后,从服务器向主服务器发送PSYNC命令.执行同步,此时主服务器也成为从服务器的客户端,这样主服务才可以在之后的操作中将缓冲区的内容发送过去.而且主从之间才可以保持同步.

## 命令传播

之后,主从服务器进入命令传播阶段.主从服务器的数据保持一致.


# 心跳检测

在命令传播阶段,从服务器会默认以每秒一次的频率向主服务发送命令 REPLCONF ACK <replication_offset>,其中replication_offset是从服务器当前的复制偏移量.

作用:

1. 检测主从服务器的网络状态
2. 辅助实现min-slaves选项
3. 检测命令丢失

## 检测主从服务器的网络状态

如果主服务器超过一秒没有收到从服务器的REPLCONF命令,则主服务器认为从服务器出现故障.

info replication命令的lag项表示从服务器上一次向从服务器发送命令距今有多少秒.

## 辅助实现min-slaves选项

redis的两个选项

````
min-slaves-to-write 3
min-slaves-to-lag 10

````
表示从服务器的数量少于三个或者三个服务器的值都大于等于10秒,主服务器将拒绝执行写命令.

## 检测命令丢失

主服务器在收到从服务器的replication_offset参数后,发现与自己的偏移量不一致,会从复制积压缓冲区中取出数据重新发送.以保证主从数据的同步.这个类似于部分重同步操作.

- redis2.8版本添加了REPLCONF ACK 命令和复制积压缓冲区,以前的版本中主从复制的命令丢失都不会被发觉.为了保证复制时主从服务器的数据一致性.最好使用2.8+版本









