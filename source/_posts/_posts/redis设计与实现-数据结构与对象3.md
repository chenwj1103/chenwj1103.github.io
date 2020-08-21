---
title: redis设计与实现-数据结构与对象3
date: 2018-01-08 23:19:04
tags: redis,数据结构与对象
categories: redis

---

- redis的数据对象

 redis不是使用前面介绍的数据结构来实现键值对数据库,而是基于这些数据结构创建了一个对象系统.包括字符串对象.列表对象.hash对象,集合对象和有序集合对象.
 
# 对象的类型与编码

Redis 使用对象来表示数据库中的键和值， 每次当我们在 Redis 的数据库中新创建一个键值对时， 我们至少会创建两个对象， 一个对象用作键值对的键**（键对象）**， 另一个对象用作键值对的值**（值对象）**

Redis 中的每个对象都由一个 redisObject 结构表示， 该结构中和保存数据有关的三个属性分别是 type 属性、 encoding 属性和 ptr 属性：

````
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 指向底层实现数据结构的指针
    void *ptr;

    // ...

} robj;

````

## 类型

对象的 type 属性记录了对象的类型



类型常量 |对象的名称 |     | 
----|--------------|----| 
REDIS_STRING | 字符串对象  | 
REDIS_LIST | 列表对象  | 
REDIS_HASH | 哈希对象  | 
REDIS_SET | 集合对象  | 
REDIS_ZSET | 有序集合对象  | 


Redis 数据库保存的键值对来说， **键总是一个字符串对象**， 而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种.

TYPE 命令的实现方式也与此类似， 当我们对一个数据库键执行 TYPE 命令时， 命令返回的结果为数据库键对应的值对象的类型

如下:

````
# 键为字符串对象，值为字符串对象

redis> SET msg "hello world"
OK

redis> TYPE msg
string

# 键为字符串对象，值为列表对象

redis> RPUSH numbers 1 3 5
(integer) 6

redis> TYPE numbers
list

````

## 编码和底层实现

对象的 ptr 指针指向对象的底层实现数据结构， 而这些数据结构由对象的 encoding 属性决定。

- encoding 属性记录了对象所使用的编码如下所示:


编码常量 |编码所对应的底层数据结构 |     | 
---------|--------------|----| 
REDIS_ENCODING_INT | long 类型的整数  | 
REDIS_ENCODING_EMBSTR | embstr 编码的简单动态字符串  | 
REDIS_ENCODING_RAW | 简单动态字符串  | 
REDIS_SET | 集合对象  | 
REDIS_ENCODING_HT | 字典  | 
REDIS_ENCODING_LINKEDLIST | 双端链表  | 
REDIS_ENCODING_ZIPLIST | 压缩列表  | 
REDIS_ENCODING_INTSET | 整数集合  | 
REDIS_ENCODING_SKIPLIST | 跳跃表和字典  | 


- 每种类型的对象都至少使用了两种不同的编码， 表 8-4 列出了每种类型的对象可以使用的编码。

类型	 |编码	|对象|
---------|--------------|----| 
REDIS_STRING	|REDIS_ENCODING_INT	|使用整数值实现的字符串对象。|
REDIS_STRING	|REDIS_ENCODING_EMBSTR	|使用 embstr 编码的简单动态字符串实现的字符串对象。|
REDIS_STRING	|REDIS_ENCODING_RAW	|使用简单动态字符串实现的字符串对象。|
REDIS_LIST	|REDIS_ENCODING_ZIPLIST	|使用压缩列表实现的列表对象。|
REDIS_LIST	|REDIS_ENCODING_LINKEDLIST	|使用双端链表实现的列表对象。|
REDIS_HASH	|REDIS_ENCODING_ZIPLIST	|使用压缩列表实现的哈希对象。|
REDIS_HASH	|REDIS_ENCODING_HT	|使用字典实现的哈希对象。|
REDIS_SET	|REDIS_ENCODING_INTSET	|使用整数集合实现的集合对象。|
REDIS_SET	|REDIS_ENCODING_HT	|使用字典实现的集合对象。|
REDIS_ZSET	|REDIS_ENCODING_ZIPLIST	|使用压缩列表实现的有序集合对象。|
REDIS_ZSET	|REDIS_ENCODING_SKIPLIST	|使用跳跃表和字典实现的有序集合对象。|

- 使用 OBJECT ENCODING 命令可以查看一个数据库键的值对象的编码：

````
redis> SET msg "hello wrold"
OK

redis> OBJECT ENCODING msg
"embstr"

redis> SET story "long long long long long long ago ..."
OK

redis> OBJECT ENCODING story
"raw"

````

# 字符串对象

字符串对象的编码可以是 int 、 raw 或者 embstr 。

## int

如果一个字符串对象保存的是**整数值**， 并且这个整数值可以**用 long 类型来表示**， 那么字符串对象会将整数值保存在字符串对象结构的 ptr 属性里面**（将 void* 转换成 long ）**， 并将字符串对象的**编码设置为 int** 。


````
redis> SET number 10086
OK

redis> OBJECT ENCODING number
"int"

````

![int编码的字符串对象](/images/redis/struct2/int编码的字符串对象.png)

## raw

如果字符串对象保存的是一个**字符串值**， 并且这个字符串值的**长度大于 39 字节**， 那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值， 并将对象的编码设置为 raw 。

````
redis> SET story "Long, long, long ago there lived a king ..."
OK

redis> STRLEN story
(integer) 43

redis> OBJECT ENCODING story
"raw"

````
![raw编码的字符串对象](/images/redis/struct2/raw编码的字符串对象.png)


## embstr

如果字符串对象保存的是一个**字符串值**， 并且这个字符串值的长度**小于等于 39 字节**， 那么字符串对象将使用 embstr 编码的方式来保存这个字符串值。

- embstr和raw的区别:

embstr 编码是专门用于保存短字符串的一种优化编码方式， 这种编码和 raw 编码一样， 都使用 redisObject 结构和 sdshdr 结构来表示字符串对象， 但 **raw 编码会调用两次内存分配函数**来分别创建 redisObject 结构和 sdshdr 结构， 而 embstr **编码则通过调用一次内存分配函数来分配一块连续的空间**， 空间中依次包含 redisObject 和 sdshdr 两个结构， 如图 8-3 所示。

![embstr编码的字符串对象](/images/redis/struct2/embstr编码的字符串对象.png)

- embstr存储的好处

1. e**mbstr 编码将创建字符串对象所需的内存分配次数从 raw 编码的两次降低为一次。
2. 释放 embstr 编码的字符串对象只需要调用一次内存释放函数， 而释放 raw 编码的字符串对象需要调用两次内存释放函数。
3. 因为 embstr 编码的字符串对象的所有数据都保存在一块连续的内存里面， 所以这种编码的字符串对象比起 raw 编码的字符串对象能够更好地利用缓存带来的优势。



## 编码的转换

int 编码的字符串对象和 embstr 编码的字符串对象在条件满足的情况下， 会被转换为 raw 编码的字符串对象。

- 例子

对于 int 编码的字符串对象来说， 如果我们向对象执行了一些命令， 使得这个对象保存的不再是整数值， 而是一个字符串值， 那么字符串对象的编码将从 int 变为 raw 

````
redis> SET number 10086
OK

redis> OBJECT ENCODING number
"int"

redis> APPEND number " is a good number!"
(integer) 23

redis> GET number
"10086 is a good number!"

redis> OBJECT ENCODING number
"raw"

````

- _注意_

 因为 Redis 没有为 embstr 编码的字符串对象编写任何相应的修改程序 （只有 int 编码的字符串对象和 raw 编码的字符串对象有这些程序）， **所以 embstr 编码的字符串对象实际上是只读的**： 当我们对 embstr 编码的字符串对象执行任何修改命令时， 程序会先将对象的编码从 embstr 转换成 raw ， 然后再执行修改命令； 因为这个原因， embstr 编码的字符串对象在执行修改命令之后， 总会变成一个 raw 编码的字符串对象。
 
_以下例子中执行append操作后,字符串的长度没有超过39,但由于embstr的只读特性._

````
redis> SET msg "hello world"
OK

redis> OBJECT ENCODING msg
"embstr"

redis> APPEND msg " again!"
(integer) 18

redis> OBJECT ENCODING msg
"raw"

````



# 列表对象

列表对象的编码可以是 ziplist 或者 linkedlist 。

## ziplist编码

ziplist 编码的列表对象使用压缩列表作为底层实现， 每个压缩列表节点（entry）保存了一个列表元素。

````
redis> RPUSH numbers 1 "three" 5
(integer) 3

````

以上的numbers对象使用的是ziplist编码,则对象将会是如下的样子.

![ziplist编码的list对象](/images/redis/struct2/ziplist编码的list对象.png)

## linkedlist编码

linkedlist 编码的列表对象使用双端链表作为底层实现， 每个双端链表节点（node）都保存了一个字符串对象， 而每个字符串对象都保存了一个列表元素。

如果numbers对象使用的是linklist编码,则对象将会时如下的样子.

![linklist编码的list对象](/images/redis/struct2/linklist编码的list对象.png)

linkedlist 编码的列表对象在底层的双端链表结构中包含了多个字符串对象， 这种嵌套字符串对象的行为在稍后介绍的哈希对象、集合对象和有序集合对象中都会出现， 字符串对象是 Redis 五种类型的对象中唯一一种会被其他四种类型对象嵌套的对象。

![完整的字符串对象表示](/images/redis/struct2/完整的字符串对象表示.png) 

以上linklist中的字符串对象只是简单的表示.



## 编码转化

当列表对象可以同时满足以下两个条件时， 列表对象使用 ziplist 编码：

1. 列表对象保存的所有字符串元素的长度都小于 64 字节；
2. 列表对象保存的元素数量小于 512 个；

以下代码展示了列表对象因为保存了长度太大的元素而进行编码转换的情况：

````
# 所有元素的长度都小于 64 字节
redis> RPUSH blah "hello" "world" "again"
(integer) 3

redis> OBJECT ENCODING blah
"ziplist"

# 将一个 65 字节长的元素推入列表对象中
redis> RPUSH blah "wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww"
(integer) 4

# 编码已改变
redis> OBJECT ENCODING blah
"linkedlist"

````

除此之外， 以下代码展示了列表对象因为保存的元素数量过多而进行编码转换的情况：

````
# 列表对象包含 512 个元素
redis> EVAL "for i=1,512 do redis.call('RPUSH', KEYS[1], i) end" 1 "integers"
(nil)

redis> LLEN integers
(integer) 512

redis> OBJECT ENCODING integers
"ziplist"

# 再向列表对象推入一个新元素，使得对象保存的元素数量达到 513 个
redis> RPUSH integers 513
(integer) 513

# 编码已改变
redis> OBJECT ENCODING integers
"linkedlist"

````


# 哈希对象

哈希对象的编码可以是 ziplist 或者 hashtable 。

## ziplist编码

ziplist 编码的哈希对象使用压缩列表作为底层实现， 每当有新的键值对要加入到哈希对象时， 程序会先将保存了键的压缩列表节点推入到压缩列表表尾， 然后再将保存了值的压缩列表节点推入到压缩列表表尾， 因此：

1. 保存了同一键值对的两个节点总是紧挨在一起， 保存键的节点在前， 保存值的节点在后；
2. 先添加到哈希对象中的键值对会被放在压缩列表的表头方向， 而后来添加到哈希对象中的键值对会被放在压缩列表的表尾方向。



````
redis> HSET profile name "Tom"
(integer) 1

redis> HSET profile age 25
(integer) 1

redis> HSET profile career "Programmer"
(integer) 1

````

以上代码如果编码是zipist,则如下表示:

![ziplist编码的hash对象](/images/redis/struct2/ziplist编码的hash对象.png)

![hash对象的压缩列表的底层实现](/images/redis/struct2/hash对象的压缩列表的底层实现.png)

另一方面， hashtable 编码的哈希对象使用字典作为底层实现， 哈希对象中的每个键值对都使用一个字典键值对来保存：

1. 字典的每个键都是一个字符串对象， 对象中保存了键值对的键；
2. 字典的每个值都是一个字符串对象， 对象中保存了键值对的值。

![hashtable编码的hash对象](/images/redis/struct2/hashtable编码的hash对象.png)


## 编码转换

当哈希对象可以同时满足以下两个条件时， 哈希对象使用 ziplist 编码：

1. 哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节；
2. 哈希对象保存的键值对数量小于 512 个；

不能同时满足这两个条件的哈希对象需要使用 hashtable 编码。

以下代码展示了哈希对象因为键值对的键长度太大而引起编码转换的情况：

````
# 哈希对象只包含一个键和值都不超过 64 个字节的键值对
redis> HSET book name "Mastering C++ in 21 days"
(integer) 1

redis> OBJECT ENCODING book
"ziplist"

# 向哈希对象添加一个新的键值对，键的长度为 66 字节
redis> HSET book long_long_long_long_long_long_long_long_long_long_long_description "content"
(integer) 1

# 编码已改变
redis> OBJECT ENCODING book
"hashtable"
````

除了键的长度太大会引起编码转换之外， 值的长度太大也会引起编码转换， 以下代码展示了这种情况的一个示例：

````
# 哈希对象只包含一个键和值都不超过 64 个字节的键值对
redis> HSET blah greeting "hello world"
(integer) 1

redis> OBJECT ENCODING blah
"ziplist"

# 向哈希对象添加一个新的键值对，值的长度为 68 字节
redis> HSET blah story "many string ... many string ... many string ... many string ... many"
(integer) 1

# 编码已改变
redis> OBJECT ENCODING blah
"hashtable"

````

最后， 以下代码展示了哈希对象因为包含的键值对数量过多而引起编码转换的情况：

````
# 创建一个包含 512 个键值对的哈希对象
redis> EVAL "for i=1, 512 do redis.call('HSET', KEYS[1], i, i) end" 1 "numbers"
(nil)

redis> HLEN numbers
(integer) 512

redis> OBJECT ENCODING numbers
"ziplist"

# 再向哈希对象添加一个新的键值对，使得键值对的数量变成 513 个
redis> HMSET numbers "key" "value"
OK

redis> HLEN numbers
(integer) 513

# 编码改变
redis> OBJECT ENCODING numbers
"hashtable"
````
     

# 集合对象

集合对象的编码可以是 intset 或者 hashtable 。

## inset编码

intset 编码的集合对象使用整数集合作为底层实现， 集合对象包含的所有元素都被保存在**整数集合**里面。

````
redis> SADD numbers 1 3 5
(integer) 3

````

![inset编码的整数集合对象](/images/redis/struct2/inset编码的整数集合对象.png)


 hashtable 编码的集合对象使用字典作为底层实现， 字典的每个键都是一个字符串对象， 每个字符串对象包含了一个集合元素， 而字典的值则全部被设置为 NULL 。
 
````
redis> SADD fruits "apple" "banana" "cherry"
(integer) 3

````

![hashtable编码的集合对象](/images/redis/struct2/hashtable编码的集合对象.png)


## 编码的转换

当集合对象可以同时满足以下两个条件时， 对象使用 intset 编码：

1. 集合对象保存的所有元素都是整数值；
2. 集合对象保存的元素数量不超过 512 个；

不能满足这两个条件的集合对象需要使用 hashtable 编码。


举个例子， 以下代码创建了一个只包含整数元素的集合对象， 该对象的编码为 intset ：

````
redis> SADD numbers 1 3 5
(integer) 3

redis> OBJECT ENCODING numbers
"intset"
不过， 只要我们向这个只包含整数元素的集合对象添加一个字符串元素， 集合对象的编码转移操作就会被执行：

redis> SADD numbers "seven"
(integer) 1

redis> OBJECT ENCODING numbers
"hashtable"

````

除此之外， 如果我们创建一个包含 512 个整数元素的集合对象， 那么对象的编码应该会是 intset ：

````
redis> EVAL "for i=1, 512 do redis.call('SADD', KEYS[1], i) end" 1 integers
(nil)

redis> SCARD integers
(integer) 512

redis> OBJECT ENCODING integers
"intset"
````

但是， 只要我们再向集合添加一个新的整数元素， 使得这个集合的元素数量变成 513 ， 那么对象的编码转换操作就会被执行：

````
redis> SADD integers 10086
(integer) 1

redis> SCARD integers
(integer) 513

redis> OBJECT ENCODING integers
"hashtable"
````

# 有序集合对象

有序集合的编码可以是 ziplist 或者 skiplist 。

## ziplist编码

ziplist 编码的有序集合对象使用压缩列表作为底层实现， 每个集合元素使用两个紧挨在一起的压缩列表节点来保存， 第一个节点保存元素的成员（member）， 而第二个元素则保存元素的分值（score）。

压缩列表内的集合元素按分值从小到大进行排序， 分值较小的元素被放置在靠近表头的方向， 而分值较大的元素则被放置在靠近表尾的方向。

````
redis> ZADD price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3

````

ziplist编码

![ziplist编码的有序集合对象](/images/redis/struct2/hashtable编码的集合对象.png)

![有序集合在压缩列表中的实现](/images/redis/struct2/有序集合在压缩列表中的实现.png)


## skiplist

skiplist 编码的有序集合对象使用 zset 结构作为底层实现， 一个 zset 结构同时包含一个字典和一个跳跃表：

````
typedef struct zset {

    zskiplist *zsl;

    dict *dict;

} zset;

````

zset 结构中的 zsl 跳跃表按分值从小到大保存了所有集合元素， 每个跳跃表节点都保存了一个集合元素： 跳跃表节点的 object 属性保存了元素的成员， 而跳跃表节点的 score 属性则保存了元素的分值。 通过这个跳跃表， **程序可以对有序集合进行范围型操作**， 比如 ZRANK 、 ZRANGE 等命令就是基于跳跃表 API 来实现的。

zset 结构中的 dict 字典为有序集合创建了一个从成员到分值的映射， 字典中的每个键值对都保存了一个集合元素： 字典的键保存了元素的成员， 而字典的值则保存了元素的分值。 通过这个字典， **程序可以用 O(1) 复杂度查找给定成员的分值， ZSCORE 命令就是根据这一特性实现的**

**注意**

值得一提的是， 虽然 zset 结构同时使用跳跃表和字典来保存有序集合元素， 但这两种数据结构都会通过指针来共享相同元素的成员和分值

![skiplist编码的有序集合](/images/redis/struct2/skiplist编码的有序集合.png)

![有序集合同时被保存在字典和skiplist中](/images/redis/struct2/有序集合同时被保存在字典和skiplist中.png)

## 编码的转换

当有序集合对象可以同时满足以下两个条件时， 对象使用 ziplist 编码：

1. 有序集合保存的元素数量小于 128 个；
2. 有序集合保存的所有元素成员的长度都小于 64 字节；

不能满足以上两个条件的有序集合对象将使用 skiplist 编码。


以下代码展示了有序集合对象因为包含了过多元素而引发编码转换的情况：

````
# 对象包含了 128 个元素
redis> EVAL "for i=1, 128 do redis.call('ZADD', KEYS[1], i, i) end" 1 numbers
(nil)

redis> ZCARD numbers
(integer) 128

redis> OBJECT ENCODING numbers
"ziplist"

# 再添加一个新元素
redis> ZADD numbers 3.14 pi
(integer) 1

# 对象包含的元素数量变为 129 个
redis> ZCARD numbers
(integer) 129

# 编码已改变
redis> OBJECT ENCODING numbers
"skiplist"

````

以下代码则展示了有序集合对象因为元素的成员过长而引发编码转换的情况：

````
# 向有序集合添加一个成员只有三字节长的元素
redis> ZADD blah 1.0 www
(integer) 1

redis> OBJECT ENCODING blah
"ziplist"

# 向有序集合添加一个成员为 66 字节长的元素
redis> ZADD blah 2.0 oooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo
(integer) 1

# 编码已改变
redis> OBJECT ENCODING blah
"skiplist"

````


# 类型检查与命令多态


## 命令的类型

一种命令可以对任何类型的键执行， 比如说 DEL 命令、 EXPIRE 命令、 RENAME 命令、 TYPE 命令、 OBJECT 命令， 等等。

另一种命令只能对特定类型的键执行

SET 、 GET 、 APPEND 、 STRLEN 等命令只能对字符串键执行；

HDEL 、 HSET 、 HGET 、 HLEN 等命令只能对哈希键执行；

RPUSH 、 LPOP 、 LINSERT 、 LLEN 等命令只能对列表键执行；

SADD 、 SPOP 、 SINTER 、 SCARD 等命令只能对集合键执行；

ZADD 、 ZCARD 、 ZRANK 、 ZSCORE 等命令只能对有序集合键执行；

## 类型检查的实现

![类型检查的实现](/images/redis/struct2/类型检查的实现.png)


## 多态命令的实现

Redis 除了会根据值对象的类型来判断键是否能够执行指定命令之外， 还会根据值对象的编码方式， 选择正确的命令实现代码来执行命令。

例如对一个list对象的建执行llen命令

1. 如果列表对象的编码为 ziplist ， 那么说明列表对象的实现为压缩列表， 程序将使用 ziplistLen 函数来返回列表的长度；
2. 如果列表对象的编码为 linkedlist ， 那么说明列表对象的实现为双端链表， 程序将使用 listLength 函数来返回双端链表的长度；

![LLEN命令执行过程](/images/redis/struct2/LLEN命令执行过程.png)


# 内存回收

Redis 在自己的对象系统中构建了一个引用计数（reference counting）技术实现的内存回收机制， 通过这一机制， 程序可以通过跟踪对象的引用计数信息， 在适当的时候自动释放对象并进行内存回收。

每个对象的引用计数信息由 redisObject 结构的 refcount 属性记录：

````
typedef struct redisObject {

    // ...

    // 引用计数
    int refcount;

    // ...

} robj;

````

对象的引用计数信息会随着对象的使用状态而不断变化：

1. 在创建一个新对象时， 引用计数的值会被初始化为 1 ；
2. 当对象被一个新程序使用时， 它的引用计数值会被增一；
3. 当对象不再被一个程序使用时， 它的引用计数值会被减一；
4. 当对象的引用计数值变为 0 时， 对象所占用的内存会被释放。

# 对象共享

除了用于实现引用计数内存回收机制之外， 对象的引用计数属性还带有对象共享的作用。

键A创建了一个保存整数值100的字符串对象.如果创建一个键B同样保存一个整数值100的字符串对象,则数据库键会执行以下操作

1. 将数据库键的值指针指向一个现有的值对象；
2. 将被共享的值对象的引用计数增一。

![被共享的字符串对象](/images/redis/struct2/被共享的字符串对象.png)

共享对象机制对于节约内存非常有帮助， 数据库中保存的相同值对象越多， 对象共享机制就能节约越多的内存。

**注意**

Redis 会在初始化服务器时， 创建一万个字符串对象， 这些对象包含了从 0 到 9999 的所有整数值， 当服务器需要用到值为 0 到 9999 的字符串对象时， 服务器就会使用这些共享对象， 而不是新创建对象。


举个例子， 如果我们创建一个值为 100 的键 A ， 并使用 OBJECT REFCOUNT 命令查看键 A 的值对象的引用计数， 我们会发现值对象的引用计数为 2 ：

````
redis> SET A 100
OK

redis> OBJECT REFCOUNT A
(integer) 2

````
引用这个值对象的两个程序分别是持有这个值对象的服务器程序， 以及共享这个值对象的键 A ， 如图 8-22 所示。

![引用计数为2的共享对象](/images/redis/struct2/引用计数为2的共享对象.png)

# 对象的空转时长

除了前面介绍过的 type 、 encoding 、 ptr 和 refcount 四个属性之外， redisObject 结构包含的最后一个属性为 lru 属性， 该属性记录了对象最后一次被命令程序访问的时间：

OBJECT IDLETIME 命令可以打印出给定键的空转时长， 这一空转时长就是通过将当前时间减去键的值对象的 lru 时间计算得出的：

````
redis> SET msg "hello world"
OK

# 等待一小段时间
redis> OBJECT IDLETIME msg
(integer) 20

# 等待一阵子
redis> OBJECT IDLETIME msg
(integer) 180

# 访问 msg 键的值
redis> GET msg
"hello world"

# 键处于活跃状态，空转时长为 0
redis> OBJECT IDLETIME msg
(integer) 0

````


























