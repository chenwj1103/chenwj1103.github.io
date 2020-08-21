---
title: 《SQL基础》SQL学习指南学习笔记二 查询入门 过滤 多表查询
date: 2017-11-10 23:22:30
tags: sql学习指南 
categories: sql

---

## 查询机制

当使用如下命令连接上数据库后，会提示你的connecttion id 是多少。

```
zhuningning@ubuntu:~$ mysql -u lrngsql -p 
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.20 MySQL Community Server (GPL)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.


```

在执行你的语句之前，要进行以下验证

- 用户是否有权限执行以下语句；
-  用户是否有权限访问目标数据；
- 语句的语法是否正确；

通过了以上测试，语句会传递给查询优化器（负责查询到最有效率的执行方式），之后执行查询。


## 查询语句

查询语句由以下子句组成

- select： 确定结果集中应该包含哪些列。
- from：指明所要提取数据的表，以及这些表时如何连接的。
- where：过滤不需要的表。
- group by ：用于对具有相同列值的行进行分组。
- having：过滤不需要的组。
- order by：按一个或者多个列，对最后的结果集中的行进行排序。


### select子句

该子句用于在所有的可能的列中，选择查询结果集包包含哪些列。
在该子句中可以包括字符串、表达式、内建函数等。

例子如下：

```
mysql> select emp_id,'active',emp_id*3.14159,upper(lname) from employee;
+--------+--------+----------------+--------------+
| emp_id | active | emp_id*3.14159 | upper(lname) |
+--------+--------+----------------+--------------+
|      1 | active |        3.14159 | SMITH        |
|      2 | active |        6.28318 | BARKER       |
|      3 | active |        9.42477 | TYLER        |
|      4 | active |       12.56636 | HAWTHORNE    |
|      5 | active |       15.70795 | GOODING      |
|      6 | active |       18.84954 | FLEMING      |
|      7 | active |       21.99113 | TUCKER       |
|      8 | active |       25.13272 | PARKER       |
|      9 | active |       28.27431 | GROSSMAN     |
|     10 | active |       31.41590 | ROBERTS      |
|     11 | active |       34.55749 | ZIEGLER      |
|     12 | active |       37.69908 | JAMESON      |
|     13 | active |       40.84067 | BLAKE        |
|     14 | active |       43.98226 | MASON        |
|     15 | active |       47.12385 | PORTMAN      |
|     16 | active |       50.26544 | MARKHAM      |
|     17 | active |       53.40703 | FOWLER       |
|     18 | active |       56.54862 | TULMAN       |
+--------+--------+----------------+--------------+
18 rows in set (0.00 sec)

```

如果只是执行内置函数，就不需要加后面的子句。

```
mysql> select version(),user(),database();
+-----------+----------+------------+
| version() | user()   | database() |
+-----------+----------+------------+
| 5.7.20    | lrngsql@ | bank       |
+-----------+----------+------------+
1 row in set (0.00 sec)


```

#### 列的别名

- 列的别名,可以在查询的字段后使用 as 或者直接在列后添加别名。为了增加可读性，建议添加as关键字。

例子如下：

```
mysql> select emp_id,'active' as active,emp_id*3.14159 as empid_x_pi,upper(lname) as last_name_upper from employee;
+--------+--------+------------+-----------------+
| emp_id | active | empid_x_pi | last_name_upper |
+--------+--------+------------+-----------------+
|      1 | active |    3.14159 | SMITH           |
|      2 | active |    6.28318 | BARKER          |
|      3 | active |    9.42477 | TYLER           |
|      4 | active |   12.56636 | HAWTHORNE       |
|      5 | active |   15.70795 | GOODING         |
|      6 | active |   18.84954 | FLEMING         |
|      7 | active |   21.99113 | TUCKER          |
|      8 | active |   25.13272 | PARKER          |
|      9 | active |   28.27431 | GROSSMAN        |
|     10 | active |   31.41590 | ROBERTS         |
|     11 | active |   34.55749 | ZIEGLER         |
|     12 | active |   37.69908 | JAMESON         |
|     13 | active |   40.84067 | BLAKE           |
|     14 | active |   43.98226 | MASON           |
|     15 | active |   47.12385 | PORTMAN         |
|     16 | active |   50.26544 | MARKHAM         |
|     17 | active |   53.40703 | FOWLER          |
|     18 | active |   56.54862 | TULMAN          |
+--------+--------+------------+-----------------+
18 rows in set (0.00 sec)

```

#### 去除重复行

去除重复行，可以使用distinct关键字。注意 ** 产生无重复结果集需要首先对数据进行排序 ,这对于大的结果是相当耗时的。因此当有需求时再使用DISTINCT关键字，否则没必要。 **


### from 子句

from子句定义了查询中所使用的表，以及连接这些表的方式。

表的概念:

- 永久表（create table语句创建的表）;

从表中查询数据时，可以在表名后加 as 别名 为表添加一个实例的别名。

- 临时表（子查询所返回的表）；

子查询可以出现在select语句中的各个部分，并且别包含在圆括号中。

例子：

```

mysql> select e.emp_id,e.fname,e.lname from (select emp_id,fname,lname,start_date,title from employee) e;
+--------+----------+-----------+
| emp_id | fname    | lname     |
+--------+----------+-----------+
|      1 | Michael  | Smith     |
|      2 | Susan    | Barker    |
|      3 | Robert   | Tyler     |
|      4 | Susan    | Hawthorne |
|      5 | John     | Gooding   |
|      6 | Helen    | Fleming   |
|      7 | Chris    | Tucker    |
|      8 | Sarah    | Parker    |
|      9 | Jane     | Grossman  |
|     10 | Paula    | Roberts   |
|     11 | Thomas   | Ziegler   |
|     12 | Samantha | Jameson   |
|     13 | John     | Blake     |
|     14 | Cindy    | Mason     |
|     15 | Frank    | Portman   |
|     16 | Theresa  | Markham   |
|     17 | Beth     | Fowler    |
|     18 | Rick     | Tulman    |
+--------+----------+-----------+

```
这是一个不是使用的例子，但可以表示临时表的用处。

- 虚拟表 （使用create view子句所创建的视图）

视图时存储在数据字典中的查询。它的行为表现的像一个表，但实际上并不是一个表。当查询视图时，该查询会被绑定到视图定义上。

视图的作用：

- 用户隐藏列，简化数据库设计。

如下例子：

```
// 创建视图
mysql> create view employee_vw  as select emp_id,fname,lname,year(start_date) start_year from employee;
Query OK, 0 rows affected (0.05 sec)

//查询视图
mysql> SELECT * from employee_vw;
+--------+----------+-----------+------------+
| emp_id | fname    | lname     | start_year |
+--------+----------+-----------+------------+
|      1 | Michael  | Smith     |       2001 |
|      2 | Susan    | Barker    |       2002 |
|      3 | Robert   | Tyler     |       2000 |
|      4 | Susan    | Hawthorne |       2002 |
|      5 | John     | Gooding   |       2003 |
|      6 | Helen    | Fleming   |       2004 |
|      7 | Chris    | Tucker    |       2004 |
|      8 | Sarah    | Parker    |       2002 |
|      9 | Jane     | Grossman  |       2002 |
|     10 | Paula    | Roberts   |       2002 |
|     11 | Thomas   | Ziegler   |       2000 |
|     12 | Samantha | Jameson   |       2003 |
|     13 | John     | Blake     |       2000 |
|     14 | Cindy    | Mason     |       2002 |
|     15 | Frank    | Portman   |       2003 |
|     16 | Theresa  | Markham   |       2001 |
|     17 | Beth     | Fowler    |       2002 |
|     18 | Rick     | Tulman    |       2002 |
+--------+----------+-----------+------------+
18 rows in set (0.01 sec)

```


### where子句

在结果集中过滤掉不需要的行。

当where子句中有多个条件时，可以使用AND或者OR进行连接。当混合使用不同的操作符时，开发者应当使用圆括号来分割成组的条件。

例子：

```
mysql> select emp_id,fname,lname,start_date,title from employee where (title ='Head Teller' AND start_date >'2003-01-01')  or (title='Teller' AND start_date >'2004-01-01');
+--------+-------+---------+------------+-------------+
| emp_id | fname | lname   | start_date | title       |
+--------+-------+---------+------------+-------------+
|      6 | Helen | Fleming | 2004-03-17 | Head Teller |
|      7 | Chris | Tucker  | 2004-09-15 | Teller      |
+--------+-------+---------+------------+-------------+
2 rows in set (0.00 sec)

```


### group by子句和having子句

此例子是表示在服务器返回结果集之前对数据再一次进行提炼。

**where group by  和having的区别 ** 

[参考](http://blog.csdn.net/Shine_rise/article/details/54934242)  该文章中最后两行是错误的，详细看下文中的执行顺序。

where：数据库中常用的是where关键字，用于在初始表中筛选查询。它是一个约束声明，用于约束数据，在返回结果集之前起作用。

group by：对select查询出来的结果集按照某个字段或者表达式进行分组（这里不能说明select语句时在group by中执行，
意思时说group by的字段必须在select的字段中存在。），获得一组组的集合，然后从每组中取出一个指定字段或者
表达式的值。 

在说group by的时候，我们还需要了解聚合函数，聚合函数是SQL语言中一种特殊的函数。例如：

- count(*)：获取数量
- sum()：求和(这里要注意求和是忽略null值的，null与其他数值相加结果为null，所以可以通过ifnull(xxx,0)将null的值赋为0）
- avg()：求平均数
- max()：求最大值
- min()：求最小值

这些函数和其它函数的根本区别就是它们一般作用在多条记录上。
我们需要注意的是：在使用group by的SQL语句中，select中返回的字段，必须满足以下两个条件之一：(不太确定，欢迎讨论)

- 包含在group by语句的后面，作为分组的依据；
- 这些字段包含在聚合函数中。

having：用于对where和group by查询出来的分组经行过滤，查出满足条件的分组结果。它是一个过滤声明，是在查询返回结果集以后对查询结果进行的过滤操作。 

**所以having的使用需要注意以下几点：**

- having只能用于group by（分组统计语句中）分组包括显示分组group by 和隐式分组 如name ='张三'；
- where 是用于在初始表中筛选查询，having用于在where和group by 结果分组中查询
- having 子句中的每一个元素也必须出现在select列表中
- having语句可以使用聚合函数，而where不使用。
- where子句中不能使用聚合函数
- 当在包含group by子句的查询中增加过滤条件时，需要考虑过滤是针对原始数据（应该放在where子句中），还是针对分组后的数据（放到having子句中）

回到开头的那个问题：当一个语句中同时含有where、group by 、having及聚集函数时，执行顺序如下：

执行where子句查找符合条件的数据；
使用group by 子句对数据进行分组；对group by 子句形成的组运行聚集函数计算每一组的值；
最后用having 子句去掉不符合条件的组，having处理的是分组数据，而不是原始数据。

需要注意的是:

- having 子句中的每一个元素也必须出现在select列表中。有些数据库例外，如oracle.
- having子句和where子句都可以用来设定限制条件以使查询结果满足一定的条件限制。
- having子句限制的是组，而不是行。where子句中不能使用聚集函数，而having子句中可以。

在group by子句中使用with rollup修改后的查询

例子：

```

mysql> select product_cd,open_branch_id,sum(avail_balance) tot_balance
    -> from account group by product_cd,open_branch_id with rollup;
+------------+----------------+-------------+
| product_cd | open_branch_id | tot_balance |
+------------+----------------+-------------+
| BUS        |              2 |     9345.55 |
| BUS        |              4 |        0.00 |
| BUS        |           NULL |     9345.55 |
| CD         |              1 |    11500.00 |
| CD         |              2 |     8000.00 |
| CD         |           NULL |    19500.00 |
| CHK        |              1 |      782.16 |
| CHK        |              2 |     3315.77 |
| CHK        |              3 |     1057.75 |
| CHK        |              4 |    67852.33 |
| CHK        |           NULL |    73008.01 |
| MM         |              1 |    14832.64 |
| MM         |              3 |     2212.50 |
| MM         |           NULL |    17045.14 |
| SAV        |              1 |      767.77 |
| SAV        |              2 |      700.00 |
| SAV        |              4 |      387.99 |
| SAV        |           NULL |     1855.76 |
| SBL        |              3 |    50000.00 |
| SBL        |           NULL |    50000.00 |
| NULL       |           NULL |   170754.46 |
+------------+----------------+-------------+
21 rows in set (0.01 sec)

```

对应着open_branch_id 为null的行即为对所有product_cd相同的行的合计。


**多个子句的执行顺序 **

SQL查询处理的步骤序号：

1. FROM <left_table>  
2.  <join_type> JOIN <right_table> 
3. ON <join_condition> 
4. WHERE <where_condition> 
5. GROUP BY <group_by_list>  （这个过程需要排序）
6. 聚合函数 {CUBE | ROLLUP} 
7. HAVING <having_condition> 
8. SELECT  
9. DISTINCT   （这个过程需要排序）
10. ORDER BY <order_by_list>
11. limit  <select_list>

以上每个步骤都会产生一个虚拟表，该虚拟表被用作下一个步骤的输入。这些虚拟表对调用者(客户端应用程序或者外部查询)不可用。只有最后一步生成的表才会会给调用者。如果没有在查询中指定某一个子句，将跳过相应的步骤。

逻辑查询处理阶段简介：

1. FROM：对FROM子句中的前两个表执行笛卡尔积(交叉联接)，生成虚拟表VT1。
2. ON：对VT1应用ON筛选器，只有那些使为真才被插入到TV2。
3. OUTER (JOIN):如果指定了OUTER JOIN(相对于CROSS JOIN或INNER JOIN)，保留表中未找到匹配的行将作为外部行添加到VT2，生成TV3。如果FROM子句包含两个以上的表，则对上一个联接生成的结果表和下一个表重复执行步骤1到步骤3，直到处理完所有的表位置。
4. WHERE：对TV3应用WHERE筛选器，只有使为true的行才插入TV4。
5. GROUP BY：按GROUP BY子句中的列列表对TV4中的行进行分组，生成TV5。
6. 聚合函数：把超组插入VT5，生成VT6。
7. HAVING：对VT6应用HAVING筛选器，只有使为true的组插入到VT7。
8. SELECT：处理SELECT列表，产生VT8。
9. DISTINCT：将重复的行从VT8中删除，产品VT9。
10. ORDER BY：将VT9中的行按ORDER BY子句中的列列表顺序，生成一个游标(VC10)。
11. limit：从VC10的开始处选择指定数量或比例的行，生成表TV11，并返回给调用者。


### order子句

用于对结果集中的原始列数据或者根据列数据计算的表达式结果进行排序。

默认的 order by 字段A，对字段A进行升序排列（ASC），如果需要对数据将序排列，需要在字段后加DESC关键字。

如下例子：需要首先根据open_emp_id升序排列，然后根据product_cd进行降序排列。

```
mysql> SELECT open_emp_id,product_cd FROM account ORDER BY open_emp_id,product_cd DESC;
+-------------+------------+
| open_emp_id | product_cd |
+-------------+------------+
|           1 | SAV        |
|           1 | MM         |
|           1 | MM         |
|           1 | CHK        |
|           1 | CHK        |
|           1 | CHK        |
|           1 | CD         |
|           1 | CD         |
|          10 | SAV        |
|          10 | SAV        |
|          10 | CHK        |
|          10 | CHK        |
|          10 | CD         |
|          10 | CD         |
|          10 | BUS        |
|          13 | SBL        |
|          13 | MM         |
|          13 | CHK        |
|          16 | SAV        |
|          16 | CHK        |
|          16 | CHK        |
|          16 | CHK        |
|          16 | CHK        |
|          16 | BUS        |
+-------------+------------+
24 rows in set (0.00 sec)


```

## 过滤

过滤查找条件可以分为 相等查找、不等条件、范围条件、成员条件和匹配条件。

其中范围查找中的 between and 查找需要注意：

1. 必须先制定范围的下限（在between后面），然后制定 范围的上限（在end后面），否则会查出空集合。
2. beeween and操作符是，包含边界值的。

成员条件可以使用or关键字或者 in、not in关键字。

匹配条件是指使用通配符查找。

1. 正好匹配一个字符。使用 ‘_’
2. 匹配任意数目的字符(包括0个)，使用'%'

LIKE 操作符用于在 WHERE 子句中搜索列中的指定模式。

```

SELECT column_name(s)
FROM table_name
WHERE column_name LIKE pattern;
例如：
	like  ‘K% ’  表示以K开头的字符串。
	like  '%D'  表示以D结尾的字符串。
	like ‘_oogle’  表示以任一一个字符开始的，然后时oogle的字符串。如google

```

## NULL关键字

表示值的缺失：

1. 没有合适的值，比如ATM机上的自助交易并不需要employee ID列。
2. 值未确定 比如在客户创建行的时候不知道他的id
3. 值未定义 比如为某个还未添加到数据库的产品创建账户。

注意：

1. 表达式可以为null，但是不能等于null 
2. 两个NUll值彼此不能判断为相等。


**使用NULL时应注意：**

- 普通的值一般都可能进行运算符操作,例如:ID列为int,所以可以这样:ID=ID+1等,但如果一列的值为null,
null+1=null,就是说null与任何运算符运算后都为null,这就是大家说的黑洞,会吃掉所有的东西.

```
update testNull
set b=b+1
where b is null

```

结论:查询后发现b的值没有变化,仍然为null.


- 普通的值可以进行"="操作,例如条件中一般都会这样出现:sUserName='张三',如果sUserName的值为null,
要想找出所有名字为null的记录时,不能这样用:sUserName=null,因为null不是一个具体的值,任何值与它比较
时都会返回false.此时可借用is null 或者是is not null.

```
select * from testNull where a=null --返回空结果集
select * from testNull where b is null --返回结果集 2 2 NULL

```

结论:说明null是不能用"="来比较,可用is null来替换


- 在用统计函数count时会不同,例如count(ID):统计记录数.当统计的记录中的包含有null值时,它会忽略null值.

```

select count(*),count(b) from testNull 它的返回值为2 1
select count(*),count(isnull(b,'')) from testNull 它的返回值为2 2

```

结论:对于列包含null 时,统计行数是可用count(*),或者是先把null值转换成对应的值再统计,例如count(isnull(b,''));

- 对于in 的影响不同.

```

select * from testNull
where b in(null) --没有任何记录

```

结论:in在查询时会忽略null的记录,查询的时候可用is not null来查询.

- 排序时顺序有不同:当使用ORDER BY时，首先呈现NULL值。如果你用DESC以降序排序，NULL值最后显示。

```
mysql> select emp_id ,fname,lname,superior_emp_id from employee  order by superior_emp_id;
+--------+----------+-----------+-----------------+
| emp_id | fname    | lname     | superior_emp_id |
+--------+----------+-----------+-----------------+
|      2 | Susan    | Barker    |            NULL |
|      3 | Robert   | Tyler     |            NULL |
|      1 | Michael  | Smith     |            NULL |
|      4 | Susan    | Hawthorne |               3 |
|     10 | Paula    | Roberts   |               4 |
|     16 | Theresa  | Markham   |               4 |
|     13 | John     | Blake     |               4 |
|      6 | Helen    | Fleming   |               4 |
|      5 | John     | Gooding   |               4 |
|      8 | Sarah    | Parker    |               6 |
|      9 | Jane     | Grossman  |               6 |
|      7 | Chris    | Tucker    |               6 |
|     11 | Thomas   | Ziegler   |              10 |
|     12 | Samantha | Jameson   |              10 |
|     14 | Cindy    | Mason     |              13 |
|     15 | Frank    | Portman   |              13 |
|     17 | Beth     | Fowler    |              16 |
|     18 | Rick     | Tulman    |              16 |
+--------+----------+-----------+-----------------+
18 rows in set (0.00 sec)

```
 
- 永远不会有什么数据等于NULL。1不等于NULL，2也一样。但NULL也不等于NULL。所以我们只能比较它“是”或“不是”。

- count(*)表示统计行数，而count(某一个字段)表示对该值的内容统计，如果它的值为null，则该列不计数。


## case when else end 条件逻辑语句

简单的说，条件逻辑语句时程序执行时从多个路径中选择其一的能。

### case表达式返回字符串的例子

```

SELECT 
    c.cust_id,
    c.fed_id,
    CASE
        WHEN c.cust_type_cd = 'I' THEN CONCAT(i.fname, ' ', i.lname)
        WHEN c.cust_type_cd = 'B' THEN b.name
        ELSE 'Unknown'
    END name
FROM
    customer c
        LEFT JOIN
    individual i ON c.cust_id = i.cust_id
        LEFT JOIN
    business b ON c.cust_id = b.cust_id;

+---------+-------------+------------------------+
| cust_id | fed_id      | name                   |
+---------+-------------+------------------------+
|      10 | 04-1111111  | jamesd 'hand           |
|      11 | 04-2222222  | Northeast Cooling Inc. |
|      12 | 04-3333333  | Superior Auto Body     |
|      13 | 04-4444444  | AAA Insurance Inc.     |
|       1 | 111-11-1111 | James Hadley           |
|       2 | 222-22-2222 | Susan Tingley          |
|       3 | 333-33-3333 | Frank Tucker           |
|       4 | 444-44-4444 | John Hayward           |
|       5 | 555-55-5555 | Charles Frasier        |
|       6 | 666-66-6666 | John Spencer           |
|       7 | 777-77-7777 | Margaret Young         |
|       8 | 888-88-8888 | Louis Blake            |
|       9 | 999-99-9999 | Richard Farley         |
+---------+-------------+------------------------+
13 rows in set (0.00 sec)

```

### case表达式返回表达式类型的例子

例子1

```

SELECT 
    c.cust_id,
    c.fed_id,
    CASE
        WHEN
            c.cust_type_cd = 'I'
        THEN
            (SELECT 
                    CONCAT(i.fname, ' ', i.lname)
                FROM
                    individual i
                WHERE
                    i.cust_id = c.cust_id)
        WHEN
            c.cust_type_cd = 'B'
        THEN
            (SELECT 
                    b.name
                FROM
                    business b
                WHERE
                    b.cust_id = c.cust_id)
        ELSE 'Unknown'
    END name
FROM
    customer c;

+---------+-------------+------------------------+
| cust_id | fed_id      | name                   |
+---------+-------------+------------------------+
|       1 | 111-11-1111 | James Hadley           |
|       2 | 222-22-2222 | Susan Tingley          |
|       3 | 333-33-3333 | Frank Tucker           |
|       4 | 444-44-4444 | John Hayward           |
|       5 | 555-55-5555 | Charles Frasier        |
|       6 | 666-66-6666 | John Spencer           |
|       7 | 777-77-7777 | Margaret Young         |
|       8 | 888-88-8888 | Louis Blake            |
|       9 | 999-99-9999 | Richard Farley         |
|      10 | 04-1111111  | jamesd 'hand           |
|      11 | 04-2222222  | Northeast Cooling Inc. |
|      12 | 04-3333333  | Superior Auto Body     |
|      13 | 04-4444444  | AAA Insurance Inc.     |
+---------+-------------+------------------------+
13 rows in set (0.00 sec)


```

例子2

查询某个表达式的结果的个数

```
SELECT 
    c.cust_id,
    c.fed_id,
    c.cust_type_cd,
    CASE (SELECT 
            COUNT(*)
        FROM
            account a
        WHERE
            a.cust_id = c.cust_id)
        WHEN 0 THEN 'none'
        WHEN 1 THEN '1'
        WHEN 2 THEN '2'
        ELSE '3+'
    END num_accounts
FROM
    customer c;

+---------+-------------+--------------+--------------+
| cust_id | fed_id      | cust_type_cd | num_accounts |
+---------+-------------+--------------+--------------+
|       1 | 111-11-1111 | I            | 3+           |
|       2 | 222-22-2222 | I            | 2            |
|       3 | 333-33-3333 | I            | 2            |
|       4 | 444-44-4444 | I            | 3+           |
|       5 | 555-55-5555 | I            | 1            |
|       6 | 666-66-6666 | I            | 2            |
|       7 | 777-77-7777 | I            | 1            |
|       8 | 888-88-8888 | I            | 2            |
|       9 | 999-99-9999 | I            | 3+           |
|      10 | 04-1111111  | B            | 2            |
|      11 | 04-2222222  | B            | 1            |
|      12 | 04-3333333  | B            | 1            |
|      13 | 04-4444444  | B            | 1            |
+---------+-------------+--------------+--------------+
13 rows in set (0.00 sec)

```

## 多表查询

### 多表查询 需要使用连接。

连接的结果可以在逻辑上看作是由SELECT语句指定的列组成的新表。
左连接与右连接的左右指的是以两张表中的哪一张为基准，它们都是外连接。
外连接就好像是为非基准表添加了一行全为空值的万能行，用来与基准表中找不到匹配
的行进行匹配。假设两个没有空值的表进行左连接，左表是基准表，
左表的所有行都出现在结果中，右表则可能因为无法与基准表匹配而出现是空值的字段


```

mysql> select e.fname,e.lname,d.name from employee e join department d on e.dept_id = d.dept_id;
+----------+-----------+----------------+
| fname    | lname     | name           |
+----------+-----------+----------------+
| Susan    | Hawthorne | Operations     |
| Helen    | Fleming   | Operations     |
| Chris    | Tucker    | Operations     |
| Sarah    | Parker    | Operations     |
| Jane     | Grossman  | Operations     |
| Paula    | Roberts   | Operations     |
| Thomas   | Ziegler   | Operations     |
| Samantha | Jameson   | Operations     |
| John     | Blake     | Operations     |
| Cindy    | Mason     | Operations     |
| Frank    | Portman   | Operations     |
| Theresa  | Markham   | Operations     |
| Beth     | Fowler    | Operations     |
| Rick     | Tulman    | Operations     |
| John     | Gooding   | Loans          |
| Michael  | Smith     | Administration |
| Susan    | Barker    | Administration |
| Robert   | Tyler     | Administration |
+----------+-----------+----------------+
18 rows in set (0.00 sec)


```

以上列子是内连接查询的结果，如果一个表中的dept_id 列中存在某个值，但这个值在另一个表的dept_id列中不存在，
那么相关行的链接会失败，在结果集中会排除包含该值的行。


- ANSI连接语法

这种旧的连接方式不包含on子句，而是在from子句中定义个表的别名。并使用逗号隔开。

它具有以下优点：

1. 连接条件和过滤条件被分割到on子句和where子句，使查询语句容易被理解。
2. 每两个表之间的连接条件都在自己的on子句中列出，这样不容易忽略这些条件。

例子：

```

mysql> SELECT e.fname,e.lname,d.name from employee e ,department d where e.dept_id =d.dept_id;
+----------+-----------+----------------+
| fname    | lname     | name           |
+----------+-----------+----------------+
| Susan    | Hawthorne | Operations     |
| Helen    | Fleming   | Operations     |
| Chris    | Tucker    | Operations     |
| Sarah    | Parker    | Operations     |
| Jane     | Grossman  | Operations     |
| Paula    | Roberts   | Operations     |
| Thomas   | Ziegler   | Operations     |
| Samantha | Jameson   | Operations     |
| John     | Blake     | Operations     |
| Cindy    | Mason     | Operations     |
| Frank    | Portman   | Operations     |
| Theresa  | Markham   | Operations     |
| Beth     | Fowler    | Operations     |
| Rick     | Tulman    | Operations     |
| John     | Gooding   | Loans          |
| Michael  | Smith     | Administration |
| Susan    | Barker    | Administration |
| Robert   | Tyler     | Administration |
+----------+-----------+----------------+
18 rows in set (0.00 sec)


```

- 连接两次使用同一张表时，可以为表取别名。别名即表的实例，数据库服务器可以区分所引用的实例。

- 自连接

一张表中存这雇员的信息和一个指向本表的外键。
此时要查询每个雇员的姓名和主管道的姓名，即可使用自连接。

例子：

```
mysql> select e.fname,e_mgr.fname as emgrName from employee e  inner join employee e_mgr on e.superior_emp_id =e_mgr.emp_id;
+----------+----------+
| fname    | emgrName |
+----------+----------+
| Susan    | Robert   |
| John     | Susan    |
| Helen    | Susan    |
| Chris    | Helen    |
| Sarah    | Helen    |
| Jane     | Helen    |
| Paula    | Susan    |
| Thomas   | Paula    |
| Samantha | Paula    |
| John     | Susan    |
| Cindy    | John     |
| Frank    | John     |
| Theresa  | Susan    |
| Beth     | Theresa  |
| Rick     | Theresa  |
+----------+----------+
15 rows in set (0.00 sec)


```

- 自连接的不等连接

每一个组中的成员与组里的别的成员进行一场象棋比赛。

```

//这样会产生重复记录，即a VS b  和b VS a
mysql> select e1.fname,e1.lname, 'VS' vs , e2.fname,e2.lname from employee e1 inner join employee e2 on e1.emp_id  !=e2.emp_id where e1.title ='Teller' and e2.title ='Teller';

// 所以需要过滤，采用>的条件

mysql> select e1.fname,e1.lname, 'VS' vs , e2.fname,e2.lname from employee e1 inner join employee e2 on e1.emp_id  >e2.emp_id where e1.title ='Teller' and e2.title ='Teller';
+----------+----------+----+----------+----------+
| fname    | lname    | vs | fname    | lname    |
+----------+----------+----+----------+----------+
| Sarah    | Parker   | VS | Chris    | Tucker   |
| Jane     | Grossman | VS | Chris    | Tucker   |
| Jane     | Grossman | VS | Sarah    | Parker   |
| Thomas   | Ziegler  | VS | Chris    | Tucker   |
| Thomas   | Ziegler  | VS | Sarah    | Parker   |
| Thomas   | Ziegler  | VS | Jane     | Grossman |
| Samantha | Jameson  | VS | Chris    | Tucker   |
| Samantha | Jameson  | VS | Sarah    | Parker   |
| Samantha | Jameson  | VS | Jane     | Grossman |
| Samantha | Jameson  | VS | Thomas   | Ziegler  |
| Cindy    | Mason    | VS | Chris    | Tucker   |
| Cindy    | Mason    | VS | Sarah    | Parker   |
| Cindy    | Mason    | VS | Jane     | Grossman |
| Cindy    | Mason    | VS | Thomas   | Ziegler  |
| Cindy    | Mason    | VS | Samantha | Jameson  |
| Frank    | Portman  | VS | Chris    | Tucker   |
| Frank    | Portman  | VS | Sarah    | Parker   |
| Frank    | Portman  | VS | Jane     | Grossman |
| Frank    | Portman  | VS | Thomas   | Ziegler  |
| Frank    | Portman  | VS | Samantha | Jameson  |
| Frank    | Portman  | VS | Cindy    | Mason    |
| Beth     | Fowler   | VS | Chris    | Tucker   |
| Beth     | Fowler   | VS | Sarah    | Parker   |
| Beth     | Fowler   | VS | Jane     | Grossman |
| Beth     | Fowler   | VS | Thomas   | Ziegler  |
| Beth     | Fowler   | VS | Samantha | Jameson  |
| Beth     | Fowler   | VS | Cindy    | Mason    |
| Beth     | Fowler   | VS | Frank    | Portman  |
| Rick     | Tulman   | VS | Chris    | Tucker   |
| Rick     | Tulman   | VS | Sarah    | Parker   |
| Rick     | Tulman   | VS | Jane     | Grossman |
| Rick     | Tulman   | VS | Thomas   | Ziegler  |
| Rick     | Tulman   | VS | Samantha | Jameson  |
| Rick     | Tulman   | VS | Cindy    | Mason    |
| Rick     | Tulman   | VS | Frank    | Portman  |
| Rick     | Tulman   | VS | Beth     | Fowler   |
+----------+----------+----+----------+----------+
36 rows in set (0.00 sec)

```

### 外连接

```

LEFT JOIN (等价于LEFT OUT JOIN) 关键字从左表（table1）返回所有的行，即使右表（table2）中没有匹配。
如果右表中没有匹配，则结果为 NULL。

例子：
SELECT  websites.name, access_log.count, access_log.date
FROM   websites  LEFT JOIN　access_log ON websites.id = access_log.site_id　ORDER BY access_log.count DESC;

+---------------+-------+------------+
| name          | count | date       |
+---------------+-------+------------+
| Facebook      |   545 | 2016-05-16 |
| Google        |   230 | 2016-05-14 |
| 菜鸟教程      |   220 | 2016-05-15 |
| Facebook      |   205 | 2016-05-14 |
| 菜鸟教程      |   201 | 2016-05-17 |
| 菜鸟教程      |   100 | 2016-05-13 |
| Google        |    45 | 2016-05-10 |
| 微博          |    13 | 2016-05-15 |
| 淘宝          |    10 | 2016-05-14 |
| stackoverflow |  NULL | NULL       |
+---------------+-------+------------+

RIGHT JOIN 与INNER JOIN的意思相同，不做过多的解释。

```

### union和union all关键字

- 要使用UNION或者UNION all 必须满足以下两个条件：

两个数据集合必须具有相同的列；
两个数据集中对应的列的数据类型必须时一致的（或者时服务器中数据类型可以转换）；

- 区别：

union对连接后的集合排序并去除重复项，而union all保留重复项。

例子：

```

mysql> select 'ind' type_cd,cust_id,lname name from individual union all select 'bus' type_cd,cust_id,name from business;
+---------+---------+------------------------+
| type_cd | cust_id | name                   |
+---------+---------+------------------------+
| ind     |       1 | Hadley                 |
| ind     |       2 | Tingley                |
| ind     |       3 | Tucker                 |
| ind     |       4 | Hayward                |
| ind     |       5 | Frasier                |
| ind     |       6 | Spencer                |
| ind     |       7 | Young                  |
| ind     |       8 | Blake                  |
| ind     |       9 | Farley                 |
| bus     |      10 | Chilton Engineering    |
| bus     |      11 | Northeast Cooling Inc. |
| bus     |      12 | Superior Auto Body     |
| bus     |      13 | AAA Insurance Inc.     |
+---------+---------+------------------------+
13 rows in set (0.00 sec)

```

- 对复合查询结果排序

如果需要对复合查询的结果进行排序，那么可以在最后一个查询后面增加order by 子句。当在order by 子句中指定要排序的列时，需要从复合查询的第一个查询中选取列名。所以，建议对两个查询的各列定义不同的别名。


### 子查询

子查询返回的结果集类型决定了它可能如何被使用。任何查询返回的数据在包含语句执行完成之后都会被丢弃，说这事的则查询像一个具有作用域的临时表。这意味着sql执行完毕，子查询结果所占用的内存将会被清空。

#### 非关联子查询

非关联子查询是指，它可以单独执行不需要引用包含语句中的任何内容。

- 如果在等式条件下使用子查询，而子查询返回多行结果，则会出错。

例子：

```
mysql> select account_id,product_cd,cust_id from account where open_emp_id <> (select e.emp_id from employee e inner join  branch b on e.assigned_branch_id =b.branch_id where e.title='Teller' AND b.city='Woburn');
ERROR 1242 (21000): Subquery returns more than 1 row

```

错误的原因时open_emp_id 不能等于结果集。


- 多行单列子查询，即多行结果可以在非等式的 IN和NOT IN 运算符中使用。

```
mysql> select branch_id ,name,city from branch where name In ('HeadQuarters','Quincy Branch');
+-----------+---------------+---------+
| branch_id | name          | city    |
+-----------+---------------+---------+
|         1 | Headquarters  | Waltham |
|         3 | Quincy Branch | Quincy  |
+-----------+---------------+---------+
2 rows in set (0.00 sec)


```

- ALL和ANY运算符（不常用。一般使用IN和NOT IN代替。）

all运算符用于将某单值与集合中的每个值比较，而any用于个结果集中的每个成员比较。与all不同的时，any运算符中，只要有一个比较成立，则条件为真。all需要每一个都成立，才为真。


例子：

```

mysql> select emp_id,fname,lname,title from employee where emp_id <> all (select superior_emp_id from employee where superior_emp_id is NOT NULL);
+--------+----------+----------+----------------+
| emp_id | fname    | lname    | title          |
+--------+----------+----------+----------------+
|      1 | Michael  | Smith    | President      |
|      2 | Susan    | Barker   | Vice President |
|      5 | John     | Gooding  | Loan Manager   |
|      7 | Chris    | Tucker   | Teller         |
|      8 | Sarah    | Parker   | Teller         |
|      9 | Jane     | Grossman | Teller         |
|     11 | Thomas   | Ziegler  | Teller         |
|     12 | Samantha | Jameson  | Teller         |
|     14 | Cindy    | Mason    | Teller         |
|     15 | Frank    | Portman  | Teller         |
|     17 | Beth     | Fowler   | Teller         |
|     18 | Rick     | Tulman   | Teller         |
+--------+----------+----------+----------------+
12 rows in set (0.00 sec)


```



#### 关联子查询

- 与非关联子查询不同，关联子查询不是在包含语句执行前执行一次，而是为每一个候选行都执行一次。

下面的例子中首先关联查询计算每个客户的账户数，接着包含查询检索出哪些拥有两个账户。

例子：

```

mysql> select c.cust_id,c.cust_type_cd,c.city from customer c where 2 =(select count(*) from account a where a.cust_id =c.cust_id);
+---------+--------------+---------+
| cust_id | cust_type_cd | city    |
+---------+--------------+---------+
|       2 | I            | Woburn  |
|       3 | I            | Quincy  |
|       6 | I            | Waltham |
|       8 | I            | Salem   |
|      10 | B            | Salem   |
+---------+--------------+---------+
5 rows in set (0.01 sec)

```

- exists运算符

如果只关心存在关系，而不在乎数量就可以使用exists关键字。

```

//以下时exists和not exists的用法。
mysql> select a.account_id ,a.product_cd,a.cust_id,a.avail_balance from account a where not exists (select 1 from transaction t where t.account_id =a.account_id and t.txn_date ='2008-09-22');
+------------+------------+---------+---------------+
| account_id | product_cd | cust_id | avail_balance |
+------------+------------+---------+---------------+
|          1 | CHK        |       1 |       1057.75 |
|          2 | SAV        |       1 |        500.00 |
|          3 | CD         |       1 |       3000.00 |
|          4 | CHK        |       2 |       2258.02 |
|          5 | SAV        |       2 |        200.00 |
|          7 | CHK        |       3 |       1057.75 |
|          8 | MM         |       3 |       2212.50 |
|         10 | CHK        |       4 |        534.12 |
|         11 | SAV        |       4 |        767.77 |
|         12 | MM         |       4 |       5487.09 |
|         13 | CHK        |       5 |       2237.97 |
|         14 | CHK        |       6 |        122.37 |
|         15 | CD         |       6 |      10000.00 |
|         17 | CD         |       7 |       5000.00 |
|         18 | CHK        |       8 |       3487.19 |
|         19 | SAV        |       8 |        387.99 |
|         21 | CHK        |       9 |        125.67 |
|         22 | MM         |       9 |       9345.55 |
|         23 | CD         |       9 |       1500.00 |
|         24 | CHK        |      10 |      23575.12 |
|         25 | BUS        |      10 |          0.00 |
|         27 | BUS        |      11 |       9345.55 |
|         28 | CHK        |      12 |      38552.05 |
|         29 | SBL        |      13 |      50000.00 |
+------------+------------+---------+---------------+
24 rows in set (0.00 sec)

mysql> select a.account_id ,a.product_cd,a.cust_id,a.avail_balance from account a where exists (select 1 from transaction t where t.account_id =a.account_id and t.txn_date ='2008-09-22');
Empty set (0.01 sec)


子查询可能返回1或者0 ，使用1代表是否至少能返回一行。

```


- update语句的关联查询

例子：

查询出每个账户的最新交易日期，然后修改账户的每一行的last_activity_date字段。

```

mysql> update account a set a.last_activity_date =(select max(t.txn_date) from transaction t where t.account_id =a.account_id);
Query OK, 19 rows affected (0.15 sec)
Rows matched: 24  Changed: 19  Warnings: 0


```

- delete语句中的关联查询

mysql中的delete语句使用关联子查询时，无论如何都不能使用表的别名。(暂时不知道原因)

例子：

```

mysql> delete from department where not exists (select 1 from employee  where employee.dept_id=department.dept_id);
Query OK, 0 rows affected (0.00 sec)

```


#### 如何使用子查询

- 子查询作为数据源

```

mysql> select d.dept_id ,d.name ,e_ent.how_many num_employees from department d inner join (select dept_id ,count(*) how_many from employee group by dept_id) e_ent on d.dept_id=e_ent.dept_id;
+---------+----------------+---------------+
| dept_id | name           | num_employees |
+---------+----------------+---------------+
|       1 | Operations     |            14 |
|       2 | Loans          |             1 |
|       3 | Administration |             3 |
+---------+----------------+---------------+
3 rows in set (0.00 sec)


```

子查询在from子句中必须是非关联的，它必须首先执行，然后一直保存在内存中直至包含查询执行完毕。

- 子查询作为过滤条件

子查询作为过滤条件出现在having条件中，不会出现在where条件中。

例子：

```

SELECT 
    open_emp_id, COUNT(*) how_many
FROM
    account
GROUP BY open_emp_id
HAVING COUNT(*) = (SELECT 
        MAX(emp_cnt.how_many)
    FROM
        (SELECT 
            COUNT(*) how_many
        FROM
            account
        GROUP BY open_emp_id) emp_cnt);

+-------------+----------+
| open_emp_id | how_many |
+-------------+----------+
|           1 |        8 |
+-------------+----------+
1 row in set (0.00 sec)

```

- 子查询作为表达式生成器

例子：

```

SELECT 
    emp.emp_id,
    CONCAT(emp.fname, ' ', emp.lname) emp_name,
    (SELECT 
            CONCAT(boss.fname, ' ', boss.lname)
        FROM
            employee boss
        WHERE
            boss.emp_id = emp.superior_emp_id) boss_name
FROM
    employee emp
WHERE
    emp.superior_emp_id IS NOT NULL
ORDER BY (SELECT 
        boss.lname
    FROM
        employee boss
    WHERE
        boss.emp_id = emp.superior_emp_id) , emp.lname;


+--------+------------------+-----------------+
| emp_id | emp_name         | boss_name       |
+--------+------------------+-----------------+
|     14 | Cindy Mason      | John Blake      |
|     15 | Frank Portman    | John Blake      |
|      9 | Jane Grossman    | Helen Fleming   |
|      8 | Sarah Parker     | Helen Fleming   |
|      7 | Chris Tucker     | Helen Fleming   |
|     13 | John Blake       | Susan Hawthorne |
|      6 | Helen Fleming    | Susan Hawthorne |
|      5 | John Gooding     | Susan Hawthorne |
|     16 | Theresa Markham  | Susan Hawthorne |
|     10 | Paula Roberts    | Susan Hawthorne |
|     17 | Beth Fowler      | Theresa Markham |
|     18 | Rick Tulman      | Theresa Markham |
|     12 | Samantha Jameson | Paula Roberts   |
|     11 | Thomas Ziegler   | Paula Roberts   |
|      4 | Susan Hawthorne  | Robert Tyler    |
+--------+------------------+-----------------+
15 rows in set (0.00 sec)


```







     




































































    