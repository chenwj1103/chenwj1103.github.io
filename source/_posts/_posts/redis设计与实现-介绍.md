---
title: redis设计与实现-介绍
date: 2017-12-20 01:10:57
tags: redis设计与实现
categories: redis

---


## 简介
*   可基于内存的、也可以持久化的日志型的、key-value形式的非关系型数据库
## 由来
*   是意大利的一家做实时统计系统的公司在使用mysql时，创始人Salvatore Sanfilippo发现mysql的性能非常低下。于2009年开发完成的redis数据库，并将该数据库开源。VMware公司从2010年赞助redis的开发。
## 特点
*   读写性能：读110000/s  写 80000/s
*   value支持的类型：string 、hash 、list 、set、 zset  、BitMap、 HyperLogLog 、 geospatial
*   所有的操作是原子操作
*   支持数据库的备份，支持主从备份（master-slave）
*   支持订阅发布、通知以及key过期等特性
## 相同产品对比
*   名称	 类型	数据存储选项	  查询类型	    附加功能
*   Redis	使用内存存储（in-memory） 的非关系数据库。字符串、列表、集合、散列表、有序集合。每种数据类型都有自己的专属命令;另外还有批量操作（bulk operation）和不完全（partial）的事务支持;发布与订阅， 主从复制（master/slave replication）， 持久化， 脚本（存储过程，stored procedure）
*   memcached	使用内存存储的键值缓存;键值之间的映射; 创建命令、读取命令、更新命令、删除命令以及其他几个命令;为提升性能而设的多线程服务器
*   MySQL	关系数据库;每个数据库可以包含多个表，每个表可以包含多个行； 可以处理多个表的视图（view）； 支持空间（spatial）和第三方扩展;SELECT 、 INSERT 、 UPDATE 、 DELETE 、函数、存储过程;支持ACID性质（需要使用InnoDB）， 主从复制和主主复制 （master/master replication）
*   PostgreSQL	关系数据库 ;每个数据库可以包含多个表， 每个表可以包含多个行； 可以处理多个表的视图；  支持空间和第三方扩展；支持可定制类型	SELECT 、 INSERT 、 UPDATE 、 DELETE 、内置函数、自定义的存储过程	支持ACID性质，主从复制， 由第三方支持的多主复制 （multi-master replication）
*   MongoDB	使用硬盘存储（on-disk）的非关系文档存储;每个数据库可以包含多个表， 每个表可以包含多个无schema （schema-less）的BSON文档;创建命令、读取命令、更新命令、删除命令、条件查询命令，等等;	支持map-reduce操作，主从复制，分片， 空间索引（spatial index）

## redis服务器命令
*   启动服务命令  ./redis-server &
*   启动客户端命令  ./redis-cli -h 127.0.0.1 -p 6379 -a password (远程启动)
*   ping 检测是否启动 如果启动返回的是pong
*   查看配置的命令 config get *   或者 info
*   保存当前数据库的数据到磁盘 save
*   恢复数据只需将备份文件（dump.rdb）放到安装目录启动服务即可（config get dir）
*   查看所有客户端 client list
*   查看数据库key的数量  dbsize
*   删除所有数据库的key  flushall



## 数据类型的基本命令

*   String   存：set key value  取：get key  一个键最大存储512M
*   hash     存：hmset  key  name chen age 27   取 hgetall key (取一组数据)  hget key filed   删除：hdel key field2
*   list     存：lpush key value    取：lrange key index0   indexN
*   set      存: sadd key member  取：smembers key (不允许重复)
*   zset     存：zadd key score member  取:zrangebyscore  key 100 10000      或 zrange key 0 1000

## keys操作命令
*   del key  删除操作
*   exists key 是否存在
*   expire key n  设置超时时间（s）
*   persist key  移除key的超时时间
*   ttl key  查看key的剩余超时时间 （-1代表永久存在）
*   rename  key newKey   修改key的名称
*   type key 返回可以的类型


## 发布订阅模式
*   订阅：subscribe channel
*   发布：publish channel "some thing"

## 事务
*   开始事务 命令入队 执行事务
*   multi    command   exec

## 安全
*   查看是否设置密码： config get requirepass
*   设置密码: config set requirepass "password"
*   登录验证: auth password

## redis分区


## java使用redis的相关操作


## redis的数据类型的内部实现

**首先Redis内部使用一个redisObject对象来表示所有的key和value。**

字段包括：数据类型（type）、编码方式（encoding）、数据指针（ptr）、虚拟内存（vm）、其他信息 （...）

`type的值`：(String，hash，list，set，sorted set)
    
`encoding` (raw,int,ht,zipmap,linkedlist,ziplist,intset)
    
`vm字段`，只有打开了Redis的虚拟内存功能，此字段才会真正的分配内存，该功能默认是关闭状态的
        
**字符串**可以被编码为 raw (常规字符串) 或者int (用字符串表示64位无符号整数这种编码方式是为了节省空间).

**列表类型**可以被编码为ziplist 或者 linkedlist. ziplist 是为了节省较小的列表空间而设计一种特殊编码方式.

**集合被编码**为 intset 或者 hashtable. intset 是为了存储数字的较小集合而设计的一种特殊编码方式.

**哈希表**可以被编码为 zipmap 或者hashtable. zipmap 是专为了较小的哈希表而设计的一种特殊编码方式

**有序集合**被编码为ziplist 或者 skiplist 格式. ziplist可以表示较小的有序集合, skiplist 表示任意大小多的有序集合.

## 基本数据类型介绍

### String：

- 常用命令：

set,get,decr,incr 等。

- 应用场景：

String是最常用的一种数据类型，普通的key/value存储都可以归为此类。

- 实现方式：

默认存储是二进制安全的字符串，被redisObject所引用，当遇到incr、decr操作时。将其转为数值型进行计算，此时redisObject的encoding类型为int

### Hash

- 常用命令：

hget,hset,hgetall 等。

- 应用场景：

根据用户id查找用户的姓名，年龄，出生地等信息。

- 存储方式 

1.key为id 将他的基本信息存到一个对象后以序列化的方式存储。 序列化发序列化成本高
2.key为id：姓名  key为id：年龄 多个key-value键值对存储。	多个键值对 不好维护
3.采用hash存储

- 实现方式

当成员较少时，为了节省内存，value的redisObject采用的encoding是zipmap；当数量增大时自动转为HashMap，此时encoding是ht这个限制可以在配置文件中指定（默认配置在redis根目录下的redis.conf中）



### List

- 常用命令：

lpush,rpush,lpop,rpop,lrange等。

- 应用场景：

Redis list的应用场景非常多，比如各种粉丝列表，不需要排序的列表

- 实现方式：

list的实现为一个双向链表，即可以支持反向查找和遍历。
如果redisObject的type成员值是REDIS_LIST类型的，则当该list的元素个数小于配置值list-max-ziplist-entries且元素值字符串的长度小于配置值list-max-ziplist-value则可以编码成 REDIS_ENCODING_ZIPLIST 类型存储，否则采用 Dict 来存储(Dict实际是Hash Table的一种实现，list采用ziplist数据结构存储数据，（默认配置在redis根目录下的redis.conf中）


### Set

- 常用命令：

sadd,spop,smembers,sunion 等。

- 应用场景：

set类似于list，但可以自动排重。当你需要一个要求元素不能重复的集合是可以使用set

- 实现方式：

当set集合中的元素为整数且元素个数小于配置set-max-intset-entries值时，使用intset数据结构存储(int16_t类型、int32_t 类型、 int64_t 类型。至于怎么选择是那种类型的数组，是根据其保存的值的取值范围来决定的).否则转化为Dict结构，Dict实际是Hash Table的一种实现


### Sorted set

- 常用命令：

zadd,zrange,zrem,zcard等

- 使用场景：

当你需要一个有序的并且不重复的集合列表，那么 可以选择sorted set数据结构

- 实现方式：

Redis sorted set的内部使用HashMap和跳跃表(SkipList)来保证数据的存储和有序，HashMap里放的是成员到score的映射，而跳跃表里存放的 是所有的成员，排序依据是HashMap里存的score,使用跳跃表的结构可以获得比较高的查找效率



## redis版本介绍

[redis版本介绍](http://book.51cto.com/art/201704/538507.htm)











