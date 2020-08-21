---
title: redis设计与实现-排序
copyright: true
date: 2018-01-23 01:44:33
tags: redis排序
categories: redis

---


# 排序 (快速排序算法)

redis的sort命令可以对列表键/集合键或者有序集合键进行排序

命令如下:

sort key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]

例子1: 对列表进行排序

````
127.0.0.1:6379> rpush numbers 5 2 1 4 2
(integer) 5
127.0.0.1:6379> LRANGE numbers 0 -1
1) "5"
2) "2"
3) "1"
4) "4"
5) "2"
127.0.0.1:6379> sort numbers
1) "1"
2) "2"
3) "2"
4) "4"
5) "5"
127.0.0.1:6379> sort numbers desc
1) "5"
2) "4"
3) "2"
4) "2"
5) "1"

````

例子2: 对集合进行排序

````
127.0.0.1:6379> sadd alphabet a b c d e f g h 
(integer) 8
127.0.0.1:6379> smembers alphabet
1) "e"
2) "d"
3) "h"
4) "a"
5) "g"
6) "f"
7) "b"
8) "c"
127.0.0.1:6379> sort alphabet alpha
1) "a"
2) "b"
3) "c"
4) "d"
5) "e"
6) "f"
7) "g"
8) "h"

````

例子3: 对有序集合进行排序,使用by字段排序

````
127.0.0.1:6379> zadd test-result 3.0 jack 3.5 peter 4.0 tom 
(integer) 3
127.0.0.1:6379> zrange test-result 0 -1
1) "jack"
2) "peter"
3) "tom"
127.0.0.1:6379> mset peter-number 1 tom-number 2 jack-number 3
OK
127.0.0.1:6379> sort test-result by *-number
1) "peter"
2) "tom"
3) "jack"

````

## sort<key>命令的实现

````
127.0.0.1:6379> rpush  numbers 3 1 2
(integer) 3
127.0.0.1:6379> sort numbers
1) "1"
2) "2"
3) "3"

````
以上排序的具体实现

1. 创建一个和 numbers 列表长度相同的数组， 该数组的每个项都是一个 redis.h/redisSortObject 结构， 如图 IMAGE_CREATE_ARRAY 所示。
![IMAGE_CREATE_ARRAY](/images/redis/luaandsort/IMAGE_CREATE_ARRAY.png)

2. 遍历数组， 将各个数组项的 obj 指针分别指向 numbers 列表的各个项， 构成 obj 指针和列表项之间的一对一关系， 如图 IMAGE_POINT_OBJ 所示。
![IMAGE_POINT_OBJ](/images/redis/luaandsort/IMAGE_POINT_OBJ.png)

3. 遍历数组， 将各个 obj 指针所指向的列表项转换成一个 double 类型的浮点数， 并将这个浮点数保存在相应数组项的 u.score 属性里面， 如图 IMAGE_SET_SCORE 所示。
![IMAGE_SET_SCORE](/images/redis/luaandsort/IMAGE_SET_SCORE.png)

4. 根据数组项 u.score 属性的值， 对数组进行数字值排序， 排序后的数组项按 u.score 属性的值从小到大排列， 如图 IMAGE_SORTED 所示。
![IMAGE_SORTED](/images/redis/luaandsort/IMAGE_SORTED.png)

- redisSortObject 结构

````
typedef struct _redisSortObject {

    // 被排序键的值
    robj *obj;

    // 权重
    union {

        // 排序数字值时使用
        double score;

        // 排序带有 BY 选项的字符串值时使用
        robj *cmpobj;

    } u;

} redisSortObject;

````
 
## ALPHA选项的实现

与sort<key> 类似,只不过排序的是字母.按照字母排序而不是数字的大小

## ASC和DESC选项的实现

sort<key> 默认升序排列(ASC), DESC选项是将序排列

## BY选项的实现

````
127.0.0.1:6379> zadd test-result 3.0 jack 3.5 peter 4.0 tom 
(integer) 3
127.0.0.1:6379> zrange test-result 0 -1
1) "jack"
2) "peter"
3) "tom"
127.0.0.1:6379> mset peter-number 1 tom-number 2 jack-number 3
OK
127.0.0.1:6379> sort test-result by *-number
1) "peter"
2) "tom"
3) "jack"

````
该例子中,只是将peter的权重键peter-number 1.0 放到u.score的位置,tom-number 2.0放到u.score的位置,jack-number 3.0放到u.score的位置,然后按照权重排序,输出到客户端.

## LIMIT offset count 选项

只是在排完序后跳过offset后,取出 count个元素

## GET选项

````
127.0.0.1:6379> sadd students "peter" "jack" "tom"
(integer) 3
127.0.0.1:6379> sort students alpha
1) "jack"
2) "peter"
3) "tom"

//设置以上三个元素的全名

127.0.0.1:6379> set peter-name "peter white"
OK
127.0.0.1:6379> set jack-name "jack snow"
OK
127.0.0.1:6379> set tom-name "tom smith"
OK

//sort命令首先对students集合进行排序,得到结果集,获取并返回键 peter-name jack-name tom-name的值
127.0.0.1:6379> sort students alpha get *-name
1) "jack snow"
2) "peter white"
3) "tom smith"

````

## store 选项

将排序结果存到store的key中

## 多个选项的执行顺序

1. 排序:在这一步,命令会使用ALPHA/ASC/或者DESC/BY这几个选项,对输入键进行排序.并得到一个排序结果集.
2. 限制排序结果集的长度.使用limit选项,对结果集的长度进行限制.
3. 获取外部键,在这一步,命令会使用GET选项,根据排序结果集中的元素,以及get选项选定的模式,查找并获取指定的键的值,并用这些值作为新的排序结果集.
4. 保存排序结果集,这这一步,命令会使用store选项,将结果集保存到指定的键上.
5. 向客户端返回结果

# LUA脚本

redis从2.6版本开始对LUA脚本的支持,redis客户端可以使用LUA脚本,直接在服务端原子性的执行多条redis命令.

基本命令包括EVAL和EVALSHA,管理命令的四个命令:SCRIPT FLUSH,SCRIPT EXISTS,SCRIPT LOAD,SCRIPT KILL 四个命令.

关于LUA脚本,暂时不学习.

