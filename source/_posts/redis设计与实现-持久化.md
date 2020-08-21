---
title: redis设计与实现-持久化
date: 2018-01-12 01:41:01
tags: redis持久化,RDB
categories: redis

---

# RDB持久化

服务器中的非空数据库以及他们的键值对统称为**数据库状态**.

redis是内存数据库,如果不将数据保存到磁盘,一旦服务器进程退出,服务器中的数据库状态也会消失不见.

RDB持久化功能生成的RDB文件时一个经过压缩的二进制文件.通过该文件还可以生成生成RDB文件时的数据库状态.

## RDB文件的创建和载入

save和bgsave命令可以生成RDB文件.

1. save命令会阻塞服务器进程,直到RDB文件创建完毕,阻塞期间,服务器不能处理任何命令请求.
2. bgsave会派生出一个子进程来创建RDB文件.这期间父进程会处理命令请求.子进程创建完成之后,会通知父进程.

RDB文件的载入时在服务器启动时自动执行的,服务器只要检测到有RDB文件存在,就会自动载入.

### 服务器状态

1. 在执行save命令的过程中,服务器会拒绝任何命令请求;
2. 在执行bgsave命令时,如果执行save和bgsave命令都会被拒绝,而执行bgrewriteaof命令时,会在bgsave命令执行完,才执行.
3. 执行bgrewriteaof命令时,执行bgsave命令会被拒绝,因为这两个命令都是采用子进程去执行.
4. 载入RDB过程中,服务器处于阻塞状态

## 自动间隔性保存

通过在redis.conf文件中配置save属性,可以让服务器自动执行bgsave命令.

设置保存条件

````
save 900 1
save 300 10
save 60 10000
````
![服务器状态中的保存条件](/images/redis/persistence/服务器状态中的保存条件.png)

### dirty计数器和lastsave属性

dirty属性记录距离上一次成功执行save和bgsave命令之后,服务器对数据库状态(所有数据库)执行了多少次修改操作;

lastsave属性时一个时间戳记录距离上一次成功执行save和bgsave命令的时间;

### 检查保存条件是否满足

服务器的周期性函数serverCron默认100毫秒检查一次,其中一次就是检查save选项的条件是否满足

![serverCron函数检查保存条件的过程](/images/redis/persistence/serverCron函数检查保存条件的过程.png)

上述时serverCom函数检查save条件的伪代码实现,只要满足一个条件就执行bgsave命令.bgsave命令执行完之后dirty属性会被置为0.

## RDB的文件结构

RDB文件的结构如下

![RDB文件的结构](/images/redis/persistence/RDB文件的结构.png)

该结构中大写表示常量,小写表示变量和数据.

1. REDIS是长度为5字节的二进制数据,程序在载入的时候通过此标识检查是否为RDB文件.
2. db_verison 是字符串类型的整数,表示的是RDB文件的版本.
3. databses表示的时0个或者多个数据库,如果数据库状态为空,则该长度为空.如果时非空,则长度根据数据的数量也会不同.
4. EOF长度为1个字节,当程序读到这个位置的时候遇到此标识,表示数据库的所有键值对已经全部载入完毕.
5. check_num时一个8字节的无符号的整数.保存的是一个检验和.是由以上的四个参数计算得出的.在载入时会通过以上四个参数得出的校验和与该参数进行对比,以此来检验RDB文件是否有出错或者损坏的情况.

![databases为空的RDB文件](/images/redis/persistence/databases为空的RDB文件.png)

### databases部分

![带有2个非空数据的RDB文件的数据结构](/images/redis/persistence/带有2个非空数据的RDB文件的数据结构.png)

每个非空的数据库都会保存成以下三个部分

![RDB文件中的数据库结构](/images/redis/persistence/RDB文件中的数据库结构.png)

1. SELECTED时一个1字节的常量,当程序读到该标识的时候,知道接下来是一个数据库号码.
2. db_number表示的是数据库编号.
3. key_value_pairs保存着该库里所有的键值对数据.

保存了0号库和3号库的完整的RDB文件的结构如下:

![完整的RDB文件的结构](/images/redis/persistence/完整的RDB文件的结构.png)

### key_value_pairs

#### 在RDB中的键值对对象

键值对在RDB文件中由type key value三个部分组成.

![不带过期时间的键值对](/images/redis/persistence/不带过期时间的键值对.png)

![带有过期时间的键值对](/images/redis/persistence/带有过期时间的键值对.png)

type表示底层编码或者对象类型,key是字符串对象,ExpireTime_MS 是一个常量告诉程序将来要读到的时一个过期时间戳.

#### value的编码

RDB文件中的每个value都是一个对象,编码类型记在type中.

- 字符串对象 (type=REDIS_RDB_TYPE_STRING)

字符串对象的编码是Redis_encoding_int,则该对象会以如下的结构保存.其中encoding会是8bit 16bit或者32bit来保存数据

![保存字符串的对象INT编码](/images/redis/persistence/保存字符串的对象INT编码.png)

字符串对象的编码是Redis_encoding_raw,则该对象会以如下的结构保存

如果字符串长度小于等于20,则字符串会被保存为原来的样子,否则会被压缩后在保存(开启rdbcompressiong选项的情况下)

![保存字符串对象raw编码](/images/redis/persistence/保存字符串对象raw编码.png)

REDIS_RDB_ENC_LZF表示采用LZF算法.compressed_len压缩后的长度,origin_len原始长度.compressed_string 压缩后的字符串

![有无压缩字符串的对比](/images/redis/persistence/有无压缩字符串的对比.png)

- 列表对象(type=REDIS_RDB_TYPE_LIST)

value值是一个编码为REDIS_ENCODING_LINKLIST编码类型的列表对象 结构如下

![linklist编码列表对象的保存结构](/images/redis/persistence/linklist编码列表对象的保存结构.png)

- 集合对象(type=REDIS_RDB_TYPE_SET)

value值是一个编码为REDIS_ENCODING_HT编码类型的集合对象 结构如下

![HT编码集合对象的保存结构](/images/redis/persistence/HT编码集合对象的保存结构.png)

- hash对象(type=REDIS_RDB_TYPE_HASH)
   
value值是一个编码为REDIS_ENCODING_HT编码类型的hash对象 结构如下
   
![HT编码hash表对象的保存结构](/images/redis/persistence/HT编码hash表对象的保存结构.png)

![更详细的HT编码hash表对象的保存结构](/images/redis/persistence/更详细的HT编码hash表对象的保存结构.png)


- 有序集合对象(type=REDIS_RDB_TYPE_ZSET)
   
value值是一个编码为REDIS_ENCODING_SKIPLISR编码类型的有序集合对象 结构如下
   
![skiplist编码的有序集合的保存结构](/images/redis/persistence/skiplist编码的有序集合的保存结构.png)

![更详细的skiplist编码的有序集合的保存结构](/images/redis/persistence/更详细的skiplist编码的有序集合的保存结构.png)

- INTSET编码的集合(type=REDIS_RDB_TYPE_SET_INTSET)

![INTSET编码的集合描述](/images/redis/persistence/INTSET编码的集合描述.png)

- ZIPLIST编码的列表/hash表/有序集合(type=REDIS_RDB_TYPE_LIST_ZIPLIST/type=REDIS_RDB_TYPE_HASH_ZIPLIST/type=REDIS_RDB_TYPE_ZSET_ZIPLIST)

![ziplist编码的RDB数据结构描述](/images/redis/persistence/ziplist编码的RDB数据结构描述.png)

## 分析RDB文件

使用od命令来分析redis服务器产生的RDB文件

实例

````
zhuningning@ubuntu:/usr/local/redis-3.2.10$ od -c dump.rdb 
0000000   R   E   D   I   S   0   0   0   7 372  \t   r   e   d   i   s
0000020   -   v   e   r 006   3   .   2   .   1   0 372  \n   r   e   d
0000040   i   s   -   b   i   t   s 300   @ 372 005   c   t   i   m   e
0000060 302   " 376   X   Z 372  \b   u   s   e   d   -   m   e   m 302
0000100 200 212  \f  \0 377   _   V   4 005 364 005 365   k
0000115

````
简单说几个参数  377代表EOF ,376代表SELECTED ,\0代表 0 


由于不经常用,如想了解具体细节,请自行翻阅其它资料.


# AOF持久化

AOF是通过保存服务器所执行的写命令来记录数据库状态的,AOF文件的所有命令均以Redis命令请求协议的格式保存.

![AOF持久化流程图](/images/redis/persistence/AOF持久化流程图.png)

## AOF持久化的实现

AOF持久化实现的实现分为三个步骤:追加/文件写入/文件同步

### 文件的追加

当AOF持久化功能打开时,服务器执行完一个写命令,会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾

![服务器缓冲区属性](/images/redis/persistence/服务器缓冲区属性.png)

### AOF文件的写入和同步

redis的服务器进程就是一个事件循环,这个循环中的文件事件负责接受客户端的命令请求,以及向客户端回复命令;时间事件负责指向serverCron这样的函数运行需要定时运行的函数,

服务器在处理文件事件时可能会指向写命令,使得一些内容被追加到aof_buf缓冲区,所以在服务器每次结束一个循环之前都会调用flushAppendOnlyFile函数.考虑是否将缓冲区中的内容写到AOF文件中

伪代码实现

![事件循环的伪代码实现](/images/redis/persistence/事件循环的伪代码实现.png)

flushAppendOnlyFile函数的行为有appendfsync选项的值来决定,以下是不同的appendfsync值产生的不同的持久化行为

![不同的appendfsync值产生的不同的持久化行为](/images/redis/persistence/不同的appendfsync值产生的不同的持久化行为.png)

现在操作系统中,在调用write函数写数据到文件的时候,会将数据保存在一个内存缓冲区里面,等到缓冲区的空间被填满,然后才将缓冲区中的内容写入到磁盘.但系统也提供了立即flush的函数.

- AOF持久化的效率和安全性

![AOF持久化的效率和安全性1](/images/redis/persistence/AOF持久化的效率和安全性1.png)

![AOF持久化的效率和安全性2](/images/redis/persistence/AOF持久化的效率和安全性2.png)

## AOF文件载入和数据还原

AOF文件中包含了重建数据库状态所需的所有写命令,所以服务器只要读入并重新执行一遍,就而已还原服务器关闭之前的数据库状态.

由于redis的命令只是在客户端上下文中执行,而载入AOF文件所使用的命令直接来源于AOF文件,而不是网络连接.所以服务器使用了一个没有网络连接的伪客户端来执行命令.具体步骤如:

![AOF文件载入过程](/images/redis/persistence/AOF文件载入过程.png)

## AOF文件的重写

AOF文件存储的是客户端的写命令,随着服务器运行时间越长,文件会越大.而且AOF文件进行数据还原所需要的时间就越多.所以需要对AOF文件进行重写.

通过重写功能,服务器会创建一个新的AOF文件来替代所有的AOF文件.新文件和原有的文件所存储的数据是一致的,但新文件的剔除了许多冗余的命令,所以新文件的体积通常会小很多.

AOF文件的重写并不需要对现有的文件进行读写操作,而是通过读取当前的数据库服务器状态来实现的.

![AOF重写的例子](/images/redis/persistence/AOF重写的例子.png)

AOF重写的伪代码实现:

![AOF重写的伪代码实现1](/images/redis/persistence/AOF重写的伪代码实现1.png)

![AOF重写的伪代码实现2](/images/redis/persistence/AOF重写的伪代码实现2.png)

![AOF重写的伪代码实现3](/images/redis/persistence/AOF重写的伪代码实现3.png)

重写后产生的AOF文件

![重写后的AOF文件](/images/redis/persistence/重写后的AOF文件.png)

重写后的AOF文件对应的数据库:

![重写后的AOF文件对应的数据库](/images/redis/persistence/重写后的AOF文件对应的数据库.png)

- 如果某个键的对应的value的值的个数大于64个,则需要拆成多条命令.

## AOF后台重写(BGREWRITEAOF功能)

由于aof_rewrite函数在有多条数据需要重写的情况下会长时间阻塞线程,由于redis是单线程,所以AOF重写期间(使用aof_rewrite函数)服务器无法处理客户顿的请求.所以redis使用子进程来重写.

子进程执行重写期间数据库仍然有写操作,所以服务器当前的状态和AOF文件所保存的文件不一致.为此redis服务器创建了一个AOF重写缓冲区,它只有在服务器创建子进程之后才会使用,当redis服务器执行完一个写命令之后,它会同事给AOF缓冲区和AOF重写缓冲区发送此命令.

在子进程执行AOF重写期间,服务器需要执行以下三个工作:

1. 执行客户端发送过来的命令
2. 执行后的写命令追加到AOF缓冲区
3. 将执行后的写命令追加到AOF重写缓冲区

这样会保证AOF缓冲区的内容会定时的写入和同步到AOF文件,现有的AOF文件会正常处理;而且从创建子进程开始所有的写命令都会被放入AOF重写缓冲区

当子进程执行完重写操作后,它会向父进程发送信号,父进程会完成以下工作:

1. 将AOF重写缓冲区中的内容写入到新的AOF文件中,以此来保证和数据库的状态一致;
2. 对新的AOF文件进行改名,原子性的覆盖现有的AOF文件,完成新旧文件的替换

![AOF后台重写过程](/images/redis/persistence/AOF后台重写过程.png)

# 系统故障处理

## 检查出错的文件

1. 无论是AOF还是快照的方式,将数据持久化到硬盘还是非常有必要的.除此之外,用户还必须对持久化的文件进行备份.最好的备份是把最新的快照或者最新重写的AOF文件备份到别的服务器上.
2. redis-check-aof和redis-check-dump是在系统发生故障后,检查AOF文件和快照文件的状态,并在有需要的情况下对文件进行修复.

## 更换故障主服务器

假设A/B两台机器都运行这Redis,其中A时主服务器,B是从服务器.突然A故障坏掉,如何使用同样安装了redis的C服务器作为新的主服务器?

非常简单:首先向B服务器发送一个save命令,让它创建一个新的快照文件.接着将快照文件发送给C,并在C上启动redis,最后让B成为C的从服务器,

````
redis-check-aof [--fix] <file-aof> 会扫描给定的AOF文件,寻找不正确或者不完整的命令,并且删除出错的命令以及位于出错命令之后的命令.
redis-check-dump <dump.rdb> 目前没有办法修复出错的快照文件

````


[另外书籍中推荐了一片redis持久化解密的比较好的博客](http://blog.nosqlfan.com/html/3813.html)