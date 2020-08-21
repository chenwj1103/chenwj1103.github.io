---
title: 高性能mysql-mysql高级特性
copyright: true
date: 2018-12-14 01:28:35
tags: mysql高级特性
categories: mysql

---

# 分区表

分区表的主要目的是将数据按照一个较粗的粒度分在不同的表中。这样做可以将相关的数据放在一起，另外如果想一次批量删除整个分区的数据也变得很方便。

- 分区的好处

1.表非常大以至于无法全部都放在内存中，或者表的最后部分有热点数据，其他均是历史数据
2.分区表的数据更容易维护，例如想批量删除大量数据可以使用清除整个分区的方式；
3.分区表的数据可以分布在不同的物理设备上，高效地利用多个硬件设备；
4.可以使用分区表来避免某些特殊的瓶颈；
5.备份和恢复独立的分区，非常大的数据集的场景下效果非常好；

- 分区的限制

1.一个表最多有1024个分区；
2.如果分区字段中有主键或者唯一索引的列，那么所有主键列和唯一索引列都必须包含进来；
3.分区表中无法使用外键约束；

## 分区表的原理

分区表由多个相关的底层表实现，这些底层表也是由句柄对象表示，所以我们也是可以直接访问各个分区，

存储引擎管理分区的各个底层表和管理普通表一样，分区表的索引只是在各个底层表上各自加上一个完全相同的索引。

### 分区表上的操作按照喜爱按的操作逻辑进行

- select update delete insert

操作一个分区表的时候，分区层先打开并锁住所有的底层表，然后确定哪个分区处理这条数据，再执行操作。

### 创建分区表

- 范围分区

````
CREATE TABLE `test1` (
  `id` char(32) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '自增主键(guid)',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `partition_key` int(8) NOT NULL COMMENT '分区键(格式:yyyyMMdd)',
  PRIMARY KEY (`id`,`partition_key`),
  UNIQUE KEY `id_UNIQUE` (`id`,`partition_key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
PARTITION BY RANGE (partition_key)
(PARTITION p0 VALUES LESS THAN (20180619) ENGINE = InnoDB,
 PARTITION p20180619 VALUES LESS THAN (20180620) ENGINE = InnoDB,
 PARTITION p20180621 VALUES LESS THAN (20180622) ENGINE = InnoDB,
 PARTITION p20180622 VALUES LESS THAN (20180623) ENGINE = InnoDB,
 PARTITION p20180623 VALUES LESS THAN (20180624) ENGINE = InnoDB);

````

- 列表分区

````
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)

PARTITION BY LIST(store_id)
    PARTITION pNorth VALUES IN (3,5,6,9,17),
    PARTITION pEast VALUES IN (1,2,10,11,19,20),
    PARTITION pWest VALUES IN (4,12,13,14,18),
    PARTITION pCentral VALUES IN (7,8,15,16)
);

````

mysql还支持键值分区、hash分区。但是并不常用。partition分区子句中可以使用各种函数，但是返回的值必须是一个整数。

# 视图

视图本身是一个虚拟表，不存放任何数据，在使用sql语句访问视图的时候，它返回的数据是从其他表中生成的。视图和表是在同一个命名空间，mysql在很多地方把这两个是同样对待，但是视图不能创建触发器，不能使用drop table命令删除视图。

````
CREATE VIEW t_user_view AS SELECT * FROM t_user WHERE delete_flag = 0;

````
实现视图的最简单方法是将select语句的结果存放到临时表中，当需要访问视图的时候，直接访问这个临时表就可以了。

## 视图的优点

1.提高代码的重用性，直接将查询的数据放到视图中，查询数据的时候简单查询视图就可以；
2.提高了安全性能。可以对不同的用户创建不同的视图；

# 外键约束

innoDB是目前mysql中唯一支持外键的内置存储引擎。使用外键是有成本的，外键通常要求每次修改数据时都要在另外一张表中多执行一次查找操作。
外键约束的主要目的是控制存储在外键表中的数据，但它还可以控制对主键表中数据的修改。但是实际中通常在业务上保证数据的完整性，而不是使用外键。

# 存储过程和函数

SQL语句需要先编译然后执行，而存储过程（Stored Procedure）是一组为了完成特定功能的SQL语句集，经编译后存储在数据库中，用户通过指定存储过程的名字并给定参数（如果该存储过程带有参数）来调用执行它

[mysql的存储过程](https://www.cnblogs.com/mark-chan/p/5384139.html)

# 查询缓存

mysql查询缓存保存查询返回的完整结果。当查询命中该缓存，mysql会立刻返回结果，跳过了解析、优化和执行阶段。

查询缓存系统会跟踪查询中涉及的每个表，如果这些表发生变化，那么和这个表相关的所有缓存数据都会失效。这种机制效率低但是实现代价较低，这对于一个非常繁忙的系统来说非常重要。

## 判断缓存命中

mysql判断缓存命中的方法很简单，缓存存放在一个引用表中，通过hash值引用。这个hash值包含如下因素：查询本身、当前要查询的数据库、客户端协议的版本等。

如果查询中包含一个不确定的函数，其实是不会存进缓存的，那么在查询缓存中是不可能找到缓存结果的。

- 打开查询缓存会对读写性能带来额外的消耗

1.读查询在开始之前必须检查是否命中缓存；
2.如果这个读查询可以被缓存，那么执行完之后，mysql如果返现查询缓存中没有这个查询，会将结果存入查询缓存，这会带来额外的系统消耗；
3.对写操作也会有影响，因为当对某个表写入数据的时候，mysql必须将对应表的所有缓存都置为失效，如果查询缓存非常大或者碎片非常多，这个操作就会带来很大的系统消耗。

综上，查询缓存会带来系统性能提升，但是如果这些额外消耗不断增加，再加上对查询缓存操作时一个加锁排他操作，消耗是不可忽视的。

## 查询缓存如何使用内存

查询缓存是完全存储在内存中的，所以在配置和使用它之前，我们需要先了解它是如何使用内存的。除了查询结果之外，需要缓存的还有狠毒别的维护相关的数据。

基本的管理维护数据结构需要40KB的内存资源，除此之外，mysql用于查询的缓存的内存被分成一个个的数据块，数据块是变长。每一个数据块中，存储了自己的类型、大小和存储的数据本身，还外加一个指向前后数据块的指针。数据块的类型有：存储查询结果、存储查询和数据表的映射、存储查询文本等。

当服务器启动的时候，先初始化查询缓存需要的内存。这些内存池初始是一个完整的空闲块，这个空闲块的大小就是你配置的查询缓存大小再减去维护元数据的数据结构所消耗的空间。

## 配置和维护查询缓存

### 参数

- query_cache_type 是否打开查询缓存。OFF、ON或demand。demand表示只有在查询语句中明确写明sql_cache的语句才可以放入查询缓存。

- query_cache_size 查询缓存使用的总内存空间，单位是字节。这个值必须是1024的整数倍。

- query_cache_min_res_unit 在查询缓存中分配内存时的最小单位。

- query_cache_limit mysql可以缓存的最大查询结果。

- query_cache_wlock_invalidate 如果某个数据表被其它的连接锁住，是否仍然从查询缓存中返回结果。默认是OFF

### 减少碎片

query_cache_min_res_unit合适的值可以帮助减少内存碎片导致的内存空间浪费。

可以使用flush query cache 完成碎片整理。这个命令会将所有的查询缓存重新排序，并将所有的空闲空间都聚集到查询缓存的一块区域上，这个命令并不会将查询缓存清空。

# 总结

![分区表和视图](/images/mysql/分区表和视图.png)

![外键存储过程事务](/images/mysql/外键存储过程事务.png)

![查询缓存](/images/mysql/查询缓存.png)


































