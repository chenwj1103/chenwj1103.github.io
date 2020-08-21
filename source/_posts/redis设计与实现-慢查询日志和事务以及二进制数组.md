---
title: redis设计与实现-慢查询日志和事务以及二进制数组
copyright: true
date: 2018-01-21 02:55:34
tags: redis事务,二进制数组,慢查询日志
categories: redis
---


# 慢查询日志

Redis 的慢查询日志功能用于记录执行时间超过给定时长的命令请求， 用户可以通过这个功能产生的日志来监视和优化查询速度。

1. slowlog-log-slower-than 选项指定执行时间超过多少微秒（1 秒等于 1,000,000 微秒）的命令请求会被记录到日志上。
2. slowlog-max-len 选项指定服务器最多保存多少条慢查询日志。( 当服务器储存的慢查询日志数量等于 slowlog-max-len 选项的值时， 服务器在添加一条新的慢查询日志之前， 会先将最旧的一条慢查询日志删除。)

代码demo:

````
//先将时间设置为0 以使得所有的命令都会记录到慢查询日志里
127.0.0.1:6379> config get slowlog-log-slower-than
1) "slowlog-log-slower-than"
2) "0"

//记录慢查询日志的条数
127.0.0.1:6379> config get slowlog-max-len
1) "slowlog-max-len"
2) "10"


//查询慢日志

127.0.0.1:6379> slowlog get
 1) 1) (integer) 9           # 日志的唯一标识符（uid）
    2) (integer) 1516475717  # 命令执行时的 UNIX 时间戳
    3) (integer) 5           # 命令执行的时长，以微秒计算
    4) 1) "slowlog"          # 命令以及命令参数
       2) "ge"
 2) 1) (integer) 8
    2) (integer) 1516475638
    3) (integer) 49
    4) 1) "config"
       2) "get"
       3) "slowlog-log-slower-than"
 3) 1) (integer) 7
    2) (integer) 1516475500
    3) (integer) 61
    4) 1) "config"
       2) "get"
       3) "slowlog-max-len"


````

## 慢查询记录的保存

服务器状态中包含了几个和慢查询日志功能有关的属性：

````
struct redisServer {

    // ...

    // 下一条慢查询日志的 ID
    long long slowlog_entry_id;

    // 保存了所有慢查询日志的链表
    list *slowlog;

    // 服务器配置 slowlog-log-slower-than 选项的值
    long long slowlog_log_slower_than;

    // 服务器配置 slowlog-max-len 选项的值
    unsigned long slowlog_max_len;

    // ...

};

````

slowlog 链表保存了服务器中的所有慢查询日志， 链表中的每个节点都保存了一个 slowlogEntry 结构， 每个 slowlogEntry 结构代表一条慢查询日志：

````
typedef struct slowlogEntry {

    // 唯一标识符
    long long id;

    // 命令执行时的时间，格式为 UNIX 时间戳
    time_t time;

    // 执行命令消耗的时间，以微秒为单位
    long long duration;

    // 命令与命令参数
    robj **argv;

    // 命令与命令参数的数量
    int argc;

} slowlogEntry;

````

## 慢查询日志的阅览和删除

伪代码实现

SLOWLOG GET 伪代码实现

````

def SLOWLOG_GET(number=None):

    # 用户没有给定 number 参数
    # 那么打印服务器包含的全部慢查询日志
    if number is None:
        number = SLOWLOG_LEN()

    # 遍历服务器中的慢查询日志
    for log in redisServer.slowlog:

        if number <= 0:
            # 打印的日志数量已经足够，跳出循环
            break
        else:
            # 继续打印，将计数器的值减一
            number -= 1

        # 打印日志
        printLog(log)
````

SLOWLOG LEN伪代码实现

````
def SLOWLOG_LEN():

    # slowlog 链表的长度就是慢查询日志的条目数量
    return len(redisServer.slowlog)

````

SLOWLOG RESET 伪代码实现

````
def SLOWLOG_RESET():

    # 遍历服务器中的所有慢查询日志
    for log in redisServer.slowlog:

        # 删除日志
        deleteLog(log)

````


例子:

````
//get
127.0.0.1:6379> slowlog get
 1) 1) (integer) 14
    2) (integer) 1516476187
    3) (integer) 6
    4) 1) "slowlog"
       2) "restt"
 2) 1) (integer) 13
    2) (integer) 1516476173
    3) (integer) 5
    4) 1) "slowlog"
       2) "len"
 3) 1) (integer) 12
    2) (integer) 1516476169
    3) (integer) 4
    4) 1) "slowlog"
       2) "len"

// len
127.0.0.1:6379> slowlog len
(integer) 10


// reset

127.0.0.1:6379> slowlog reset
OK

````

## 添加新日志

在每次执行命令的之前和之后， 程序都会记录微秒格式的当前 UNIX 时间戳， 这两个时间戳之间的差就是服务器执行命令所耗费的时长， 服务器会将这个时长作为参数之一传给 slowlogPushEntryIfNeeded 函数， 而 slowlogPushEntryIfNeeded 函数则负责检查是否需要为这次执行的命令创建慢查询日志， 以下伪代码展示了这一过程：

伪代码实现

````
# 记录执行命令前的时间
before = unixtime_now_in_us()

# 执行命令
execute_command(argv, argc, client)

# 记录执行命令后的时间
after = unixtime_now_in_us()

# 检查是否需要创建新的慢查询日志
slowlogPushEntryIfNeeded(argv, argc, before-after)

````

slowlogPushEntryIfNeeded函数检查slowlog-log-slower-than和slowlog-max-len这两个参数及做出相应的操作.

# 监视器

通过执行 MONITOR 命令， 客户端可以将自己变为一个监视器， 实时地接收并打印出服务器当前处理的命令请求的相关信息：

每当一个客户端向服务器发送一条命令请求时， 服务器除了会处理这条命令请求之外， 还会将关于这条命令请求的信息发送给所有监视器

实例:

````
//监视器对象客户端
127.0.0.1:6379> MONITOR
OK
1516477049.121659 [0 127.0.0.1:58004] "ping"  //在1516477049.121659个时间点,根据 IP 为 127.0.0.1 、端口号为 56604 的客户端发送的命令请求， 对 0 号数据库执行命令ping
1516477085.685457 [0 127.0.0.1:58004] "set" "testkey" "test"
1516477099.400328 [0 127.0.0.1:58004] "get" "testkey"

//执行命令的客户端
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> set testkey test
OK
127.0.0.1:6379> get testkey
"test"


````

## 成为监视器

伪代码实现

````
def MONITOR():

    # 打开客户端的监视器标志
    client.flags |= REDIS_MONITOR

    # 将客户端添加到服务器状态的 monitors 链表的末尾
    server.monitors.append(client)

    # 向客户端返回 OK
    send_reply("OK")

````


## 向监视器发送命令信息

服务器在每次处理命令请求之前， 都会调用 replicationFeedMonitors 函数， 由这个函数将被处理命令请求的相关信息发送给各个监视器。

replicationFeedMonitors 函数的伪代码:

````
def replicationFeedMonitors(client, monitors, dbid, argv, argc):

    # 根据执行命令的客户端、当前数据库的号码、命令参数、命令参数个数等参数
    # 创建要发送给各个监视器的信息
    msg = create_message(client, dbid, argv, argc)

    # 遍历所有监视器
    for monitor in monitors:

        # 将信息发送给监视器
        send_message(monitor, msg)

````

# 事务

redis通过multi/exec/watch等命令来实现事务(transaction)功能.

实例demo:

````
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set name "practical common listp"
QUEUED
127.0.0.1:6379> get name
QUEUED
127.0.0.1:6379> set author "peter seibel"
QUEUED
127.0.0.1:6379> get author
QUEUED
127.0.0.1:6379> exec
1) OK
2) "practical common listp"
3) OK
4) "peter seibel"

````

## 事务的实现

事务的实现会经历三个阶段:
1. 事务开始;
2. 命令入队;
3. 事务执行

### 事务开始

````
redis> MULTI
OK
````

MULTI 命令可以将执行该命令的客户端从非事务状态切换至事务状态， 这一切换是通过在客户端状态的 flags 属性中打开 REDIS_MULTI 标识来完成的

伪代码实现

````
def MULTI():

    # 打开事务标识
    client.flags |= REDIS_MULTI

    # 返回 OK 回复
    replyOK()

````

### 命令入队

当一个客户端处于非事务状态时， 这个客户端发送的命令会立即被服务器执行,

当一个客户端切换到事务状态之后， 服务器会根据这个客户端发来的不同命令执行不同的操作:
1. 如果客户端发送的命令为 EXEC 、 DISCARD 、 WATCH 、 MULTI 四个命令的其中一个， 那么服务器立即执行这个命令。
2. 如果客户端发送的命令是 EXEC 、 DISCARD 、 WATCH 、 MULTI 四个命令以外的其他命令， 那么服务器并不立即执行这个命令， 而是将这个命令放入一个事务队列里面， 然后向客户端返回 QUEUED 回复。

### 事务队列

Redis 客户端都有自己的事务状态， 这个事务状态保存在客户端状态的 mstate 属性里面

事务状态的状态结构:

````
typedef struct redisClient {

    // ...

    // 事务状态
    multiState mstate;      /* MULTI/EXEC state */

    // ...

} redisClient;

````

事务队列的结构:

````
typedef struct multiState {

    // 事务队列，FIFO 顺序
    multiCmd *commands;

    // 已入队命令计数
    int count;

} multiState;

````
事务队列是一个 multiCmd 类型的数组， 数组中的每个 multiCmd 结构都保存了一个已入队命令的相关信息:

````
typedef struct multiCmd {

    // 参数
    robj **argv;

    // 参数数量
    int argc;

    // 命令指针
    struct redisCommand *cmd;

} multiCmd;

````

![IMAGE_TRANSACTION_STATE事务状态](/images/redis/transactionbitmap/IMAGE_TRANSACTION_STATE事务状态.png)

### 执行事务

当一个处于事务状态的客户端向服务器发送 EXEC 命令时， 这个 EXEC 命令将立即被服务器执行： 服务器会遍历这个客户端的事务队列， 执行队列中保存的所有命令， 最后将执行命令所得的结果全部返回给客户端。

EXEC 命令的实现原理可以用以下伪代码来描述：

````
def EXEC():

    # 创建空白的回复队列
    reply_queue = []

    # 遍历事务队列中的每个项
    # 读取命令的参数，参数的个数，以及要执行的命令
    for argv, argc, cmd in client.mstate.commands:

        # 执行命令，并取得命令的返回值
        reply = execute_command(cmd, argv, argc)

        # 将返回值追加到回复队列末尾
        reply_queue.append(reply)

    # 移除 REDIS_MULTI 标识，让客户端回到非事务状态
    client.flags &= ~REDIS_MULTI

    # 清空客户端的事务状态，包括：
    # 1）清零入队命令计数器
    # 2）释放事务队列
    client.mstate.count = 0
    release_transaction_queue(client.mstate.commands)

    # 将事务的执行结果返回给客户端
    send_reply_to_client(client, reply_queue)

````

## WATCH 命令的实现

WATCH 命令是一个乐观锁,它可以在exec命令执行之前.监视任意数量的数据库键,并在exec命令执行时,检查被监视的键是否至少有一个已经被修改过了,如果是的话,服务器拒绝执行事务,并向服务器端返回代表事务执行失败的空回复.

![watch命令执行失败的例子](/images/redis/transactionbitmap/watch命令执行失败的例子.png)

在T5时执行exec,服务器会发现watch监视的键"name" 已经发生了变化,则拒绝执行A的事务.

### 使用watch命令监听数据库键

每个redis数据库保存着一个watched_keys字典,这个字典的键是某个被watch命令监视的数据库键,值则是一个链表,链表记录了所监视的相应数据库键的客户端.

以下是watched_keys字典的实例.通过watch关键字可以使得key和客户端在watched_keys字典中进行关联.

![一个watched_keys字典](/images/redis/transactionbitmap/一个watched_keys字典.png)


### 监视机制的触发

对数据库进行写操作的时候,都会调用touchWatchKey函数进行检查,检查是否客户端正在监视刚刚被命令修改过的数据库键.如果有的话,那么该函数会将监视被修改键的客户端的REDIS_DIRTY_CAS 标示打开,标示客户端的事务的安全性被破坏.

### 判断事务是否安全

在服务器收到一个客户端发过来的exec命令时,服务器检查这个客户端是否打开 REDIS_DIRTY_CAS.打开了则放弃执行,没打开则执行.

### 一个完整事务的执行过程

一个C1的客户端在执行watch name命令后,watch_keys字典的当前状态为name--C1;接下来客户端C1执行如下命令

````
C1>multi
OK

C1>set name "peter"
QUEUED

````
这时,C2客户端执行set name "test" 命令,C2执行这个set命令会导致那些正在监视name键的客户端的REDIS_DIRTY_CAS标示被打开,其中包括C1客户端

之后,C1执行exec操作,发现REDIS_DIRTY_CAS处于打开的状态,则服务器拒绝它提交事务.


## 事务的 ACID 性质

事务的原子性,一致性,隔离性和持久性简称ACID可以检查事务的可靠性和安全性.

### 原子性

要么全部执行,要么不执行.下面的例子中事务有错误的命令,所以事务中的所有命令不可以执行.


````
127.0.0.1:6379> multi 
OK
127.0.0.1:6379> get name
QUEUED
127.0.0.1:6379> get
(error) ERR wrong number of arguments for 'get' command
127.0.0.1:6379> get message
QUEUED
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> 

````

redis的事务和传统的关系型数据库最大区别在于不支持事务的回滚机制.

### 一致性

事务的一致性是指,数据库的状态在事务执行之前和执行之后(无论是否成功)都是一致的.

#### 入队错误

在入队的过程中出现错误,则redis拒绝执行该事务.

````
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set name test
QUEUED
127.0.0.1:6379> ddd
(error) ERR unknown command 'ddd'
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> 

````

#### 执行错误

比如一个type为String类型的键,执行了lpush操作.在入队时是发现不了的.

````
127.0.0.1:6379> type name
string
127.0.0.1:6379> multi
OK
127.0.0.1:6379> lpush name 1 2 3
QUEUED
127.0.0.1:6379> get name
QUEUED
127.0.0.1:6379> exec
1) (error) WRONGTYPE Operation against a key holding the wrong kind of value
2) "test"

````

出错的命令会被服务器识别出来,不去执行,所以不会对数据库造成影响.


### 隔离性

事务的隔离性是指,即使数据库中有多个事务并发的执行,各个事务之间也不会互相的影响.并且并发状态下执行的事务和串行执行的事务产生的结果完全相同.

由于redis是单线程的,所以事务的执行都是串行的,所以事务是具有隔离性的.

### 持久性

事务的持久性是指,当一个事务执行完毕时,执行这个事务的结果已经被永久的保存到数据库中了,即使服务器停机,结果也不会丢失.

redis事务只是简单的将命令包裹在一起,并没有提供持久化的功能.具体是否持久化到硬盘是有redis的持久化机制决定的.所以redis是否持久化是与redis事务没有关系的.



# 二进制位数组

redis提供了SETBIT GETBIT BITCOUNT BITOP四个命令用于处理二进制输入(位数组)

setbit命令用于为位数组指定偏移量上的二进制位设置值,位数组的**偏移量从0开始计数**,而**二进制位的值则可以是0或者1**.返回该位置原先的值.

例子:

````
127.0.0.1:6379> setbit bit 0 1
(integer) 0
127.0.0.1:6379> setbit bit 3 1
(integer) 0
127.0.0.1:6379> setbit bit 0 0
(integer) 1
````

getbit则用于获取位数组指定偏移量上的二进制位的值.

例子:

````
127.0.0.1:6379> getbit bit 100
(integer) 0
127.0.0.1:6379> getbit bit 0
(integer) 0
127.0.0.1:6379> getbit bit 3
(integer) 1
````

bitcount 用于统计位数组里面,值为1的二进制位的数量.

例子:

````
127.0.0.1:6379> bitcount bit 0 -1
(integer) 1
127.0.0.1:6379> 
````

bitcount 命令可以对多个位数组进行按位与(and) 或(or) 异或(xor)运算


与运算：0&0=0;  0&1=0;   1&0=0;    1&1=1;(两位同时为“1”，结果才为“1”，否则为0)
或运算: 0|0=0；  0|1=1；  1|0=1；   1|1=1；(参加运算的两个对象只要有一个为1，其值为1。)
异或运算:0^0=0；  0^1=1；  1^0=1；   1^1=0；(参加运算的两个对象，如果两个相应位为“异”（值不同），则该位结果为1，否则为0。)

例子:

````
127.0.0.1:6379> setbit x 3 1   x =0000 1011
(integer) 0
127.0.0.1:6379> setbit x 1 1
(integer) 0
127.0.0.1:6379> setbit x 0 1
(integer) 0
127.0.0.1:6379> setbit y 2 1   y =0000 0110
(integer) 0
127.0.0.1:6379> setbit y 1 1
(integer) 0
127.0.0.1:6379> setbit z 2 1   z =0000 0101
(integer) 0
127.0.0.1:6379> setbit z 0 1
(integer) 0
127.0.0.1:6379> bitop and resultand x y z   0000 0000
(integer) 1
127.0.0.1:6379> bitcount resultand 0  -1
(integer) 0
127.0.0.1:6379> bitop or resultor x y z     0000 1111
(integer) 1
127.0.0.1:6379> bitcount resultor 0 -1
(integer) 4
127.0.0.1:6379> bitop xor resultxor x y z  0000 1000
(integer) 1
127.0.0.1:6379> bitcount resultxor 0 -1
(integer) 1


````

## 位数组的表示

位数组采用字符串对象表示.

![SDS表示的位数组](/images/redis/transactionbitmap/SDS表示的位数组.png)

redisObject.type的值为REDIS_STRING

sdshdr.len的值为1

buf数组中的buf[0]表示存了一字节畅的位数组.

buf数组中的buf[1]字节保存了SDS程序自动追加到值末尾的空字符串'\0'

![一字节长的位数组的SDS表示](/images/redis/transactionbitmap/一字节长的位数组的SDS表示.png)

## GETBIT命令的实现

getbit <bitarray> <offset>

1. 计算byte=(offset/8) byte值记录了offset偏移量指定的二进制位保存在数组的哪个字节;
2. 计算bit=(offset mod 8) +1 ,bit位记录了offset偏移量指定的二进制位是byte字节的第几个二进制位;
3. 根据byte值和bit值.在位数组bitarray中定位offset偏移量制定的二进制位.并返回这个位的值;

例子:

GETBIT <bitarray> 3

1. 3/8 的值为0.

2. (3 mod 8) +1的值为4

3. 定位到buf[0]字节上面.人后去除第四个二进制位的值为1 返回给客户端

以上操作都在常数时间内完成,时间复杂度为O(1)

## SETBIT命令的实现

SET <bitarray> <offset> <value>

1. len =(offset/8)+1 计算出需要多少字节;
2. 检查bitarray键保存的长度是否小于len,如果是的话则扩展空间为len字节,并将所有的新空间的二进制位设置为0
3. 计算byte=(offset/8) byte值记录了offset偏移量指定的二进制位保存的位数在哪个字节上
4. bit= (offset mod 8)+1 bit记录了偏移量指定的二进制位是byte字节的第几个二进制位;
5. byte值和bit值,在bitarray键保存的位数组中定位offset偏移量指定的二进制位,先将保存新值,然后返回旧值给客户端.

## BITCOUNT命令的实现

bitcount <bitarray>

计算二进制数组中1的个数

算法:

采用的是查表和variable-precisionSWAR两种算法.具体细节可自行查阅资料.

## BITOP命令

采用的是计算机的位运算符操作,将结果放到给定的键上





















