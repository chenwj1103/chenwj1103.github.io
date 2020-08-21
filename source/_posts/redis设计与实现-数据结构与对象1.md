---
title: redis设计与实现-数据结构与对象1
date: 2017-12-26 00:43:48
tags: redis数据结构与对象
categories: redis 

---

# 简单动态字符串

- 概要

 redis没有直接使用C语言传统的字符串表示(以空字符结尾的字符数组),自己构建了一种SDS的抽象类型,即redis的默认字符串类型表示.
 
## SDS的定义

- 如图所示:

 ![SDS示例](/images/redis/struct1/SDS示例.png)
 
1.free 属性的值为 0 ， 表示这个 SDS 没有分配任何未使用空间。

2.len 属性的值为 5 ， 表示这个 SDS 保存了一个五字节长的字符串。

3.buf 属性是一个 char 类型的数组， 数组的前五个字节分别保存了 'R' 、 'e' 、 'd' 、 'i' 、 's' 五个字符， 而最后一个字节则保存了空字符 '\0' 。

`说明`

SDS 遵循 C 字符串以空字符结尾的惯例， 保存空字符的 1 字节空间不计算在 SDS 的 len 属性里面， 并且为空字符分配额外的 1 字节空间， 以及添加空字符到字符串末尾等操作都是由 SDS 函数自动完成的， 所以这个空字符对于 SDS 的使用者来说是完全透明的。

- 下图中与上图不同的是,free属性的值为5

 ![SDS示例](/images/redis/struct1/带有未使用空间的SDS示例.png)

## SDS与C字符串的区别

C语言使用N+1长度的字符数组来表示长度为N的字符串.

### 常数复杂度获取字符串长度

1. 因为 C 字符串并不记录自身的长度信息， 所以为了获取一个 C 字符串的长度， 程序必须遍历整个字符串， 对遇到的每个字符进行计数， 直到遇到代表字符串结尾的空字符为止， 这个操作的复杂度为 O(N) 。

2. 和 C 字符串不同， 因为 SDS 在 len 属性中记录了 SDS 本身的长度， 所以获取一个 SDS 长度的复杂度仅为 O(1) 。


### 杜绝缓冲区溢出

 当 SDS API 需要对 SDS 进行修改时， API 会先检查 SDS 的空间是否满足修改所需的要求， 如果不满足的话， API 会自动将 SDS 的空间扩展至执行修改所需的大小， 然后才执行实际的修改操作， 所以使用 SDS 既不需要手动修改 SDS 的空间大小， 也不会出现前面所说的缓冲区溢出问题。
 

### 减少修改字符串时带来的内存重分配次数

c字符串内存再次分配:

如果程序执行的是增长字符串的操作， 比如拼接操作（append）， 那么在执行这个操作之前， 程序需要先通过内存重分配来扩展底层数组的空间大小 —— 如果忘了这一步就会产生缓冲区溢出。

如果程序执行的是缩短字符串的操作， 比如截断操作（trim）， 那么在执行这个操作之后， 程序需要通过内存重分配来释放字符串不再使用的那部分空间 —— 如果忘了这一步就会产生内存泄漏。

SDS内存再次分配:

为了避免 C 字符串的这种缺陷， SDS 通过未使用空间解除了字符串长度和底层数组长度之间的关联： 在 SDS 中， buf 数组的长度不一定就是字符数量加一， 数组里面可以包含未使用的字节， 而这些字节的数量就由 SDS 的 free 属性记录。

策略如:空间预分配 惰性空间释放

### 二进制安全

  所有 SDS API 都会以处理二进制的方式来处理 SDS 存放在 buf 数组里的数据， 程序不会对其中的数据做任何限制、过滤、或者假设 —— 数据在写入时是什么样的， 它被读取时就是什么样。

### 兼容部分 C 字符串函数

虽然 SDS 的 API 都是二进制安全的， 但它们一样遵循 C 字符串以空字符结尾的惯例： 这些 API 总会将 SDS 保存的数据的末尾设置为空字符， 并且总会在为 buf 数组分配空间时多分配一个字节来容纳这个空字符， 这是为了让那些保存文本数据的 SDS 可以重用一部分 

### 总结

 ![SDS与C字符串的区别](/images/redis/struct1/SDS与C字符串的区别.jpg)
 
 
# 链表
 链表提供了高效的节点重排能力， 以及顺序性的节点访问方式， 并且可以通过增删节点来灵活地调整链表的长度。列表键的底层实现之一就是链表： 当一个列表键包含了数量比较多的元素， 又或者列表中包含的元素都是比较长的字符串时， Redis 就会使用链表作为列表键的底层实现。

## 链表和链表节点的实现

链表节点的结构

````
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;

````

多个 listNode 可以通过 prev 和 next 指针组成双端链表

 ![双端链表](/images/redis/struct1/双端链表.png)
 

虽然仅仅使用多个 listNode 结构就可以组成链表， 但使用 adlist.h/list 来持有链表的话,即redis进行了封装,结构如下:

````
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 链表所包含的节点数量
    unsigned long len;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

} list;

````

如下图是是一个list结构和三个listNode结构组成的链表

 ![list结构和双端链表](/images/redis/struct1/list结构和双端链表.png)
 
- Redis 的链表实现的特性可以总结如下：
 
1. 双端： 链表节点带有 prev 和 next 指针， 获取某个节点的前置节点和后置节点的复杂度都是 O(1) 。
2. 无环： 表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL ， 对链表的访问以 NULL 为终点。
3. 带表头指针和表尾指针： 通过 list 结构的 head 指针和 tail 指针， 程序获取链表的表头节点和表尾节点的复杂度为 O(1) 。
4. 带链表长度计数器： 程序使用 list 结构的 len 属性来对 list 持有的链表节点进行计数， 程序获取链表中节点数量的复杂度为 O(1) 。
5. 多态： 链表节点使用 void* 指针来保存节点值， 并且可以通过 list 结构的 dup 、 free 、 match 三个属性为节点值设置类型特定函数， 所以链表可以用于保存各种不同类型的值。

- 总结

1. 链表被广泛用于实现 Redis 的各种功能， 比如列表键， 发布与订阅， 慢查询， 监视器， 等等。
2. 每个链表节点由一个 listNode 结构来表示， 每个节点都有一个指向前置节点和后置节点的指针， 所以 Redis 的链表实现是双端链表。
3. 每个链表使用一个 list 结构来表示， 这个结构带有表头节点指针、表尾节点指针、以及链表长度等信息。
4. 因为链表表头节点的前置节点和表尾节点的后置节点都指向 NULL ， 所以 Redis 的链表实现是无环链表。
5. 通过为链表设置不同的类型特定函数， Redis 的链表可以用于保存各种不同类型的值。

# 字典

 字典可以表示数据库， 字典还是哈希键的底层实现之一.

## 字典的实现

Redis 的字典使用哈希表作为底层实现， 一个**哈希表**里面可以有多个**哈希表节点**， 而每个哈希表节点就保存了字典中的一个**键值对**。

### hash表

 hash表的结构如下所示

````

typedef struct dictht {

    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;

````

1. **table 属性是一个数组**， 数组中的每个元素都是一个指向 dict.h/dictEntry 结构的指针， 每个 dictEntry 结构保存着一个键值对。

2. size 属性记录了哈希表的大小， 也即是 table 数组的大小， 而 used 属性则记录了哈希表目前已有节点（键值对）的数量。

3. sizemask 属性的值总是等于 size - 1 ， 这个属性和哈希值一起决定一个键应该被放到 table 数组的哪个索引上面。

如下为一个空的hash表

 ![空hash表](/images/redis/struct1/空hash表.png)
 
 
### hash表节点

````

typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;


````

其中next 属性是指向另一个哈希表节点的指针， 这个指针可以将多个哈希值相同的键值对连接在一次， 以此来解决键冲突（collision）的问题。


 ![next指针将两个索引值相同的键连接在一起](/images/redis/struct1/next指针将两个索引值相同的键连接在一起.png)
 
 
### 字典

结构如下

````
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

} dict;

````

type 属性和 privdata 属性是针对不同类型的键值对， 为创建多态字典而设置的：

type 属性是一个指向 dictType 结构的指针， 每个 dictType 结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。
而 privdata 属性则保存了需要传给那些类型特定函数的可选参数。


````

typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);

    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;

````

ht 属性是一个包含两个项的数组， 数组中的每个项都是一个 dictht 哈希表， 一般情况下， 字典只使用 ht[0] 哈希表， ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。
除了 ht[1] 之外， 另一个和 rehash 有关的属性就是 rehashidx ： 它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 -1 。


普通状态下的字典.

 ![展示了一个普通状态下的字典](/images/redis/struct1/展示了一个普通状态下的字典.png)
 

## hash算法

当要将一个新的键值对添加到字典里面时， 程序需要先根据键值对的键计算出哈希值和索引值， 然后再根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。

使用字典设置的哈希函数，计算键 key 的哈希值

hash = dict->type->hashFunction(key);

使用哈希表的 sizemask 属性和哈希值，计算出索引值,根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]

index = hash & dict->ht[x].sizemask;


 ![hash算法](/images/redis/struct1/hash算法.png)

当字典被用作数据库的底层实现， 或者哈希键的底层实现时， Redis 使用 MurmurHash2 算法来计算键的哈希值。


## 解决键冲突

### 名词解释

当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时， 我们称这些键发生了冲突（collision）。

### 解决办法:

Redis 的哈希表使用链地址法（separate chaining）来解决键冲突： 每个哈希表节点都有一个 next 指针， 多个哈希表节点可以用 next 指针构成一个单向链表， 被分配到同一个索引上的多个节点可以用这个单向链表连接起来， 这就解决了键冲突的问题。

 ![解决键冲突](/images/redis/struct1/解决键冲突.png)


## rehash

### 原因

随着操作的不断执行， 哈希表保存的键值对会逐渐地增多或者减少， 为了让哈希表的负载因子（load factor）维持在一个合理的范围之内， 当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。


### Redis 对字典的哈希表执行 rehash 的步骤如下
 
1.为字典的 ht[1] 哈希表分配空间， 这个哈希表的空间大小取决于要执行的操作， 以及 ht[0] 当前包含的键值对数量 （也即是 ht[0].used 属性的值）：
  
  如果执行的是扩展操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n （2 的 n 次方幂）；
  
  如果执行的是收缩操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n 。
  
2.将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 ht[1] 哈希表的指定位置上。

3.当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后 （ht[0] 变为空表）， 释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备。



### 例子

对图 4-8 所示字典的 ht[0] 进行`扩展`操作


1.ht[0].used 当前的值为 4 ， 4 * 2 = 8 ， 而 8 （2^3）恰好是第一个大于等于 4 的 2 的 n 次方， 所以程序会将 ht[1] 哈希表的大小设置为 8 

 ![rehash之前的字典](/images/redis/struct1/rehash之前的字典.png)
 
 ![为字典的ht[1]分配表空间](/images/redis/struct1/为字典的ht[1]分配表空间.png)
 
2.将 ht[0] 包含的四个键值对都 rehash 到 ht[1] ， 如图 4-10 所示。 

![移动键值对](/images/redis/struct1/移动键值对.png)

3.释放 ht[0] ，并将 ht[1] 设置为 ht[0] ，然后为 ht[1] 分配一个空白哈希表，如图 4-11 所示。

![rehash之后的字典](/images/redis/struct1/rehash之后的字典.png)


### 哈希表的扩展与收缩

**当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作**

1.服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 1 ；
2.服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 5 ；

`负载因子的计算公式`:

负载因子 = 哈希表已保存节点数量 / 哈希表大小

load_factor = ht[0].used / ht[0].size

**当哈希表的负载因子小于 0.1 时， 程序自动开始对哈希表执行收缩操作。**

## 渐进式 rehash

### 名词解释
扩展或收缩哈希表需要将 ht[0] 里面的所有键值对 rehash 到 ht[1] 里面， 但是， 这个 rehash 动作并不是一次性、集中式地完成的， 而是分多次、渐进式地完成的。

### 渐进式rehash的步骤
1.为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
2.在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 `0` ， 表示 rehash 工作正式开始。**rehashidx代表当前rehash的索引**
3.在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， `程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1]` ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值增一。
4.随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为` -1` ， 表示 rehash 操作已完成。

### 渐进式 rehash 执行期间的哈希表操作

因为在进行渐进式 rehash 的过程中， 字典会同时使用 ht[0] 和 ht[1] 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 ht[0] 里面进行查找， 如果没找到的话， 就会继续到 ht[1] 里面进行查找， 诸如此类。

另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。






 
 
 


 


 


  
 
 