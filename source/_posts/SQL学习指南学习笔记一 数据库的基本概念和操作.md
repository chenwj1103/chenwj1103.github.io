---
title: 《SQL基础》SQL学习指南学习笔记一　数据库的基本概念和操作
date: 2017-11-03 01:07:37
tags: sql学习指南
categories: sql

---

## 基本术语

1. 实体：数据库用户所关注的对象。
2. 列：存储在表中的独立的数字片段。
3. 行：所有列的一个集合，完整的描述了一个实体或者实体上的某个行为，也称之为记录。
4. 表：行的集合，既可以存在内存中，也可以持久化到硬盘上。
5. 结果集：未持久化表的另一个名字，一般为SQL的查询结果。
6. 主键：用于唯一标识表中的每个行的一个或者多个列。
7. 外键：一个或者多个用于识别其他表中的某一个行的列。

## 使用mysql命令行工具
- 使用命令行工具可以制定用户名和想要连接的数据库

```

zhuningning@ubuntu:~$ mysql -u lrngsql -p bank
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 5.7.20 MySQL Community Server (GPL)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

```
出现如上的结果表示连接成功。



## 表的创建

- 以下为创建表person的语句:

```

mysql> create table person(
    -> person_id smallint unsigned,
    -> fname varchar(20),
    -> lname varchar(20),
    -> gender enum('M','F'),
    -> birth_date date,
    -> street varchar(20),
    -> ciry varchar(20),
    -> state varchar(20),
    -> country varchar(20),
    -> postal_code varchar(20),
    -> constraint pk_person primary key(person_id)
    -> );

```

- 注意以上的gender　enum 类型，可以约束赋值只能为Ｍ和Ｆ；

- desc person该语句可以显示表结构；
- 其中`null`表示用于各种不能赋值的情况；

```

mysql> desc person;
+-------------+----------------------+------+-----+---------+-------+
| Field       | Type                 | Null | Key | Default | Extra |
+-------------+----------------------+------+-----+---------+-------+
| person_id   | smallint(5) unsigned | NO   | PRI | NULL    |       |
| fname       | varchar(20)          | YES  |     | NULL    |       |
| lname       | varchar(20)          | YES  |     | NULL    |       |
| gender      | enum('M','F')        | YES  |     | NULL    |       |
| birth_date  | date                 | YES  |     | NULL    |       |
| street      | varchar(20)          | YES  |     | NULL    |       |
| ciry        | varchar(20)          | YES  |     | NULL    |       |
| state       | varchar(20)          | YES  |     | NULL    |       |
| country     | varchar(20)          | YES  |     | NULL    |       |
| postal_code | varchar(20)          | YES  |     | NULL    |       |
+-------------+----------------------+------+-----+---------+-------+

```

- 以下语句可以展示创建表的语句

```

mysql> show create table person;
+--------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table  | Create Table                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
+--------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| person | CREATE TABLE `person` (
  `person_id` smallint(5) unsigned NOT NULL AUTO_INCREMENT,
  `fname` varchar(20) DEFAULT NULL,
  `lname` varchar(20) DEFAULT NULL,
  `gender` enum('M','F') DEFAULT NULL,
  `birth_date` date DEFAULT NULL,
  `street` varchar(20) DEFAULT NULL,
  `ciry` varchar(20) DEFAULT NULL,
  `state` varchar(20) DEFAULT NULL,
  `country` varchar(20) DEFAULT NULL,
  `postal_code` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`person_id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=latin1 |
+--------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)


```


- 接下来创建favorite_food表

```

mysql> create table favorite_food(
    -> person_id smallint unsigned,
    -> food varchar(20),
    -> constraint pk_favorite_food primary key(person_id,food),
    -> constraint pk_fav_food_person_id foreign key(person_id) references person(person_id)
    -> );

```

- 其中pk_fav_food_person_id为外键　references person表的person_id，pk_favorite_food为联合主键，他们都属于约束。



## 操作与修改表

- 修改主键为自动增长

```

mysql> alter table person modify person_id smallint unsigned auto_increment;
Query OK, 0 rows affected (0.82 sec)

mysql> desc person;
+-------------+----------------------+------+-----+---------+----------------+
| Field       | Type                 | Null | Key | Default | Extra          |
+-------------+----------------------+------+-----+---------+----------------+
| person_id   | smallint(5) unsigned | NO   | PRI | NULL    | auto_increment |
| fname       | varchar(20)          | YES  |     | NULL    |                |
| lname       | varchar(20)          | YES  |     | NULL    |                |
| gender      | enum('M','F')        | YES  |     | NULL    |                |
| birth_date  | date                 | YES  |     | NULL    |                |
| street      | varchar(20)          | YES  |     | NULL    |                |
| ciry        | varchar(20)          | YES  |     | NULL    |                |
| state       | varchar(20)          | YES  |     | NULL    |                |
| country     | varchar(20)          | YES  |     | NULL    |                |
| postal_code | varchar(20)          | YES  |     | NULL    |                |
+-------------+----------------------+------+-----+---------+----------------+

 
```

-  插入数据操作

```

mysql> insert into person(person_id,fname,lname,gender,birth_date) values(null,'william','turnar','M','1972-05-27');
Query OK, 1 row affected (0.06 sec)

```

注意：插入的birth_date　字段的值为字符串类型，mysql数据库会将字符串自动解析为date类型。


- 查询语句


```

mysql> select * from person;
+-----------+---------+--------+--------+------------+--------+------+-------+---------+-------------+
| person_id | fname   | lname  | gender | birth_date | street | ciry | state | country | postal_code |
+-----------+---------+--------+--------+------------+--------+------+-------+---------+-------------+
|         1 | william | turnar | M      | 1972-05-27 | NULL   | NULL | NULL  | NULL    | NULL        |
+-----------+---------+--------+--------+------------+--------+------+-------+---------+-------------+
1 row in set (0.00 sec)

```

注意，生产环境中，最好不要使用 *,这样会加重解析器的负担，使得查询的效率变低。


- 更新语句

```

mysql> update person set street='beijing' where person_id=1;
Query OK, 1 row affected (0.04 sec)
Rows matched: 1  Changed: 1  Warnings: 0

```

- 删除语句

```

mysql> delete from person where person_id =1;
Query OK, 1 row affected (0.07 sec)

```

注意 不要忘记where子句，否者会删除表中所有的数据。


## 导致错误的语句

- 插入的主键不唯一;

```

mysql> insert into person(person_id,fname,lname,gender,birth_date) values(1,'william','turnar','M','1972-05-27');
ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'

```

- 插入不存在的外键;

```

mysql> insert into favorite_food values(22222,'noodles');
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint
 fails (`bank`.`favorite_food`, CONSTRAINT `favorite_food_ibfk_1` FOREIGN KEY (`person_id`) REFERENCES `person` (`person_id`))

```

由于favorite_food表中的person_id是依赖于person表的，可以将favorite_food看做是person表的子表。

- 列值不合法

```

mysql> update person set gender ='X' where person_id =1;
ERROR 1265 (01000): Data truncated for column 'gender' at row 1

```

gender是枚举类，只接受f 和ｍ

- 无效的日期类型

```

mysql> update person set birth_date ='DEC-21-1990' where person_id =2;
ERROR 1292 (22007): Incorrect date value: 'DEC-21-1990' for column 'birth_date' at row 1

```

可以使用日期转化函数 str_to_date

```

update person set birth_date=str_to_date('DEC-21-1990','%b-%d-%Y’) where person_id =2;

```

## mysql的数据类型
### 字符型数据
*字符类型数据可以分为２种：*

+ 定长的char类型，它需要使用空格向右填充； max  255byte
+ 变长的字符串 varchar类型，不需要向右填充，并且所有的字节数可变。max 65535bye　即 ６４ｋ
+ 当定义一个字符列时，必须制定该列所能存放的字符串的最大长度。

*字符集：*

+ mysql可以使用各种字符集来存储数据，使用`show character set;` 可以查看字符集列表，默认的安装数据库时，lantin1字符集会被
默认的选为字符集。
+ 可以使用以下语句，在创建数据库的时候制定字符集；

```

mysql> create database foreign_sales character set utf8;

```
*文本数据*

+ 当需要存储的数据大于64KB(VARCHAR的上限)　就要使用文本类型：
- tinytext(255 byte) 
- text(65535 byte) 
- mediumtext(16777215 byte) 
- longtext(4294967295 byte) 

`需要注意`：
- 如果装载的长度超过该类型的最大长度，数据将会被阶段；
- 文本不会消除数据的尾部空格；

### 数值型数据

*整数类型 :* 　

- tinyint : 0-255;
- smallint : 0-65535(六万);
- mediumint : 0-26777215 (一千六百万);
- int : 0-42亿;
- bigint : 0　~ ;

*浮点类型：*

- float(p,s)
- double(p,s)

### 时间数据

时间类型分为:
date    year     time     datetime     timestamp
其中后２个是相同的格式，只不过timestamp的范围为　1970-01-01 00:00:00至2037-12-31 23:59:59 且mysql中可以生成timestamp
类型的时间数据。

- [具体的数据类型参考手册](http://www.runoob.com/sql/sql-datatypes.html)
- [java数据类型和mysql数据类型对应表](http://www.cnblogs.com/jerrylz/p/5814460.html)	


### 字符串相关的函数

- 当插入的字符串的长度超过指定的字段的长度时，服务器会抛出异常。这是mysql服务器的默认操作，可以修改其配置，改成截取字符串后插入，不抛出任何异常。

- 由于mysql中的字符串使用单引号分割，所以字符串中包含单引号时可以使用 ‘\’或者单引号作为转义，即两个单引号。

-  length()函数返回字符串的个数,这个获取长度时会删除char类型的尾端的空格。

```

+---------+--------------+------------+-------------+
| cust_id | name         | state_id   | incorp_date |
+---------+--------------+------------+-------------+
|      10 | jamesd 'hand | 12-345-678 | 1995-05-01  |
+---------+--------------+------------+-------------+
1 row in set (0.00 sec)

mysql> select length(name) from business where cust_id=10;
+--------------+
| length(name) |
+--------------+
|           12 |
+--------------+
1 row in set (0.00 sec)


```

- REGEXP操作符（模式匹配）

```

mysql> select cust_id,fed_id,fed_id REGEXP '.{3}-.{2}-.{4}' isFomart from customer;
+---------+-------------+----------+
| cust_id | fed_id      | isFomart |
+---------+-------------+----------+
|       1 | 111-11-1111 |        1 |
|       2 | 222-22-2222 |        1 |
|       3 | 333-33-3333 |        1 |
|       4 | 444-44-4444 |        1 |
|       5 | 555-55-5555 |        1 |
|       6 | 666-66-6666 |        1 |
|       7 | 777-77-7777 |        1 |
|       8 | 888-88-8888 |        1 |
|       9 | 999-99-9999 |        1 |
|      10 | 04-1111111  |        0 |
|      11 | 04-2222222  |        0 |
|      12 | 04-3333333  |        0 |
|      13 | 04-4444444  |        0 |
+---------+-------------+----------+
13 rows in set (0.00 sec)

```

- concat() 返回字符串的字符串函数

```

mysql> select concat(fname,' ',lname,' has been a title  ','since ',start_date) emp_narrative from employee where title ='Teller';
+-----------------------------------------------------+
| emp_narrative                                       |
+-----------------------------------------------------+
| Chris Tucker has been a title  since 2004-09-15     |
| Sarah Parker has been a title  since 2002-12-02     |
| Jane Grossman has been a title  since 2002-05-03    |
| Thomas Ziegler has been a title  since 2000-10-23   |
| Samantha Jameson has been a title  since 2003-01-08 |
| Cindy Mason has been a title  since 2002-08-09      |
| Frank Portman has been a title  since 2003-04-01    |
| Beth Fowler has been a title  since 2002-06-29      |
| Rick Tulman has been a title  since 2002-12-12      |
+-----------------------------------------------------+
9 rows in set (0.00 sec)


```

- SubString(字段，1，end) - 从某个文本字段提取字符



### 数字相关的函数

- mod() 求余操作

```

mysql> select mod(22.75,5);
+--------------+
| mod(22.75,5) |
+--------------+
|         2.75 |
+--------------+

```

- ceil()和floor()向上和向下取整操作

- round()四舍五入操作

- ROUND(X,D)： 返回参数X的四舍五入的有 D 位小数的一个数字。如果D为0，结果将没有小数点或小数部分。

```

mysql> select round(1.287,1) as result;
+--------+
| result |
+--------+
|    1.3 |
+--------+
1 row in set (0.00 sec)

```

- 处理有符号数

sign()函数余额为负数时返回-1，为0时返回0，为正数时返回1。

```

mysql> select account_id,sign(avail_balance),abs(avail_balance) from account;
+------------+---------------------+--------------------+
| account_id | sign(avail_balance) | abs(avail_balance) |
+------------+---------------------+--------------------+
|          1 |                   1 |            1057.75 |
|          2 |                   1 |             500.00 |
|          3 |                   1 |            3000.00 |
|          4 |                   1 |            2258.02 |
|          5 |                   1 |             200.00 |
|          7 |                   1 |            1057.75 |
|          8 |                   1 |            2212.50 |
|         10 |                   1 |             534.12 |
|         11 |                   1 |             767.77 |
|         12 |                   1 |            5487.09 |
|         13 |                   1 |            2237.97 |
|         14 |                   1 |             122.37 |
|         15 |                   1 |           10000.00 |
|         17 |                   1 |            5000.00 |
|         18 |                   1 |            3487.19 |
|         19 |                   1 |             387.99 |
|         21 |                   1 |             125.67 |
|         22 |                   1 |            9345.55 |
|         23 |                   1 |            1500.00 |
|         24 |                   1 |           23575.12 |
|         25 |                   0 |               0.00 |
|         27 |                   1 |            9345.55 |
|         28 |                   1 |           38552.05 |
|         29 |                   1 |           50000.00 |
+------------+---------------------+--------------------+



```

- AVG() - 返回平均值

```
语法：
SELECT AVG(column_name) FROM table_name

例子：
从 "access_log" 表的 "count" 列获取平均值：

mysql> select AVG(count) as countASAverage from access_log;
+----------------+
| countASAverage |
+----------------+
|       174.3333 |


选择访问量高于平均访问量的 "site_id" 和 "count"：
mysql> select site_id,count from access_log where count>(select AVG(count) from access_log);
+---------+-------+
| site_id | count |
+---------+-------+
|       1 |   230 |
|       5 |   205 |
|       3 |   220 |
|       5 |   545 |
|       3 |   201 |
+---------+-------+

```





### 日期相关的函数

-  当一个字段为date   dateTime  time timestamp类型的时候，插入一个数据为字符串类型时，则服务器会将字符串转变为该类型。

- str_to_date()格式化函数

```
mysql> update individual set birth_date =STR_TO_DATE('September 17,2008','%M %d, %Y') where cust_id =1;
Query OK, 1 row affected (0.04 sec)

```


-  current_date(),current_time(),current_timestamp() 时间函数

```

mysql> select current_date(),current_time(),current_timestamp();
+----------------+----------------+---------------------+
| current_date() | current_time() | current_timestamp() |
+----------------+----------------+---------------------+
| 2017-11-29     | 00:26:22       | 2017-11-29 00:26:22 |
+----------------+----------------+---------------------+
1 row in set (0.00 sec)


```


- date_add()函数

```

//第二个参数包含了3个参数，分别是interval关键字，数量，和时间间隔类型

mysql> select date_add(current_date(),interval 5 day);
+-----------------------------------------+
| date_add(current_date(),interval 5 day) |
+-----------------------------------------+
| 2017-12-04                              |
+-----------------------------------------+
1 row in set (0.00 sec)

type 参数可以是下列值：
Type 值
MICROSECOND
SECOND
MINUTE
HOUR
DAY
WEEK
MONTH
QUARTER
YEAR
SECOND_MICROSECOND
MINUTE_MICROSECOND
MINUTE_SECOND
HOUR_MICROSECOND
HOUR_SECOND
HOUR_MINUTE
DAY_MICROSECOND
DAY_SECOND
DAY_MINUTE
DAY_HOUR
YEAR_MONTH

```

-  last_day()函数

```

mysql> select last_day('2008-09-17');
+------------------------+
| last_day('2008-09-17') |
+------------------------+
| 2008-09-30             |
+------------------------+
1 row in set (0.00 sec)

mysql> select last_day(now());
+-----------------+
| last_day(now()) |
+-----------------+
| 2017-11-30      |
+-----------------+
1 row in set (0.00 sec)

```


- dayname() 返回某一天时星期几

```

mysql> select dayname('2008-03-27');
+-----------------------+
| dayname('2008-03-27') |
+-----------------------+
| Thursday              |
+-----------------------+
1 row in set (0.00 sec)


```

- extract()截取日期的函数

```

mysql> select extract(year from now());
+--------------------------+
| extract(year from now()) |
+--------------------------+
|                     2017 |
+--------------------------+
1 row in set (0.00 sec)


```


-  datediff() 返回两个时间的像个的天数 （距今比较远的为第一个参数，则返回整数）

```

mysql> select datediff('2011-05-03','2010-08-03');
+-------------------------------------+
| datediff('2011-05-03','2010-08-03') |
+-------------------------------------+
|                                 273 |
+-------------------------------------+
1 row in set (0.00 sec)


```

- date常用函数

```

函数			描述
NOW()		返回当前的日期和时间
CURDATE()	返回当前的日期
CURTIME()	返回当前的时间
DATE()		提取日期或日期/时间表达式的日期部分
EXTRACT()	返回日期/时间的单独部分
DATE_ADD()	向日期添加指定的时间间隔
DATE_SUB()	从日期减去指定的时间间隔
DATEDIFF()	返回两个日期之间的天数
DATE_FORMAT()	用不同的格式显示日期/时间

mysql> select 
    -> DATE_FORMAT(NOW(),'%b %d %Y %h:%i %p') as date1 ,
    -> DATE_FORMAT(NOW(),'%m-%d-%Y') as date2,
    -> DATE_FORMAT(NOW(),'%d %b %y') as date3,
    -> DATE_FORMAT(NOW(),'%d %b %y') as date4;

+----------------------+------------+-----------+-----------+
| date1                | date2      | date3     | date4     |
+----------------------+------------+-----------+-----------+
| Nov 05 2017 11:05 PM | 11-05-2017 | 05 Nov 17 | 05 Nov 17 |
+----------------------+------------+-----------+-----------+


```








