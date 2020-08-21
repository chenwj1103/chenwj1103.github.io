---
title: 《SQL基础》SQL学习指南学习笔记三 索引 事务 约束
date: 2017-12-03 23:02:39
tags: sql 学习指南
categories: sql

---


# SQL 约束（Constraints）

## 概述
SQL约束用于规定表中的数据规则，如果存在违反约束的数据行为，行为将会被约束终止；

约束可以在创建表时规定，或者在创建表之后通过alter table 语句。

如下:

```	
语法：
CREATE TABLE table_name
(
column_name1 data_type(size) constraint_name,
column_name2 data_type(size) constraint_name,
column_name3 data_type(size) constraint_name,

);

例子：
CREATE TABLE www (
    id INT(11) NOT NULL AUTO_INCREMENT,
    name VARCHAR(25) NOT NULL DEFAULT 'defaultName',
    createTime DATETIME NOT NULL,
    PRIMARY KEY (id)
)  ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=UTF8;

```
## 约束的种类

### NOT NULL，非空约束

指示某列不能存储 NULL 值。（如果不向字段添加值，就无法插入新记录或者更新记录）
例子：

```

CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255)
)

```


### UNIQUE:唯一约束

保证某列的每行必须有唯一的值。（如果字段中添加的值和库里该字段中的值重复，则无法插入或者更新）

例子1：

```
mysql>create table tb2(
    tb2_id int unique,
    tb2_name varchar(20),
    tb2_age int,
    unique(tb2_name)
);
Query OK, 0 rows affected (0.40 sec)

mysql> insert into tb2(tb2_id,tb2_name,tb2_age) values(1,'张三',20);
Query OK, 1 row affected (0.05 sec)

例子2：
//违反唯一约束
mysql> insert into tb2 values(2,'张三',25);
ERROR 1062 (23000): Duplicate entry '张三' for key 'tb2_name'

例子3：
//建表时，创建约束，有约束名
mysql> create table tb3( tb3_id int,tb3_name varchar(20),tb3_age int, constraint no_id unique (tb3_id));
Query OK, 0 rows affected (0.33 sec)

insert into tb3 values (1,'张三',20);
insert into tb3(tb3_id,tb3_age) values(2,24);
select * from tb3;

//已经有了tb3_id为1的行记录，再次插入，违反唯一约束
mysql> insert into tb3(tb3_id,tb3_name,tb3_age) values(1,'李四','26');
ERROR 1062 (23000): Duplicate entry '1' for key 'no_id'

//给tb3表添加主键约束，主键名为：pk_id
alter table tb3 add constraint pk_id primary key (tb3_id);

//给tb3_name添加唯一约束
alter table tb3 add constraint un_name unique (tb3_name);

//已存在姓名为张三的记录，违反唯一约束
mysql> insert into tb3 values(3,'张三',29);
ERROR 1062 (23000): Duplicate entry '张三' for key 'un_name'

删除约束
mysql 删除约束的语句，使用index
alter table tb3 drop index un_name;

```


### PRIMARY KEY: 主键约束

NOT NULL 和 UNIQUE 的结合。确保某列（或两个列多个列的结合）有唯一标识，有助于更容易更快速地找到表中的一个特定的记录。

特点：PRIMARY KEY 约束唯一标识数据库表中的每条记录。
		主键必须包含唯一的值。
		主键列不能包含 NULL 值。
		每个表都应该有一个主键，并且每个表只能有一个主键。

例子：

```

CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
PRIMARY KEY (P_Id)
)

需要制定约束名称的，且可以制定多个列的主键约束。

CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
CONSTRAINT pk_PersonID PRIMARY KEY (P_Id,LastName)
)

添加主键约束:
ALTER TABLE Persons
ADD PRIMARY KEY (P_Id)
如需命名 PRIMARY KEY 约束，并定义多个列的 PRIMARY KEY 约束，请使用下面的 SQL 语法：
ALTER TABLE Persons ADD CONSTRAINT pk_personId  PRIMARY KEY (P_id,lastName)

删除主键约束：
ALTER TABLE Persons
DROP  PRIMARY KEY

```

### FOREIGN KEY: 外键约束

保证一个表中的数据匹配另一个表中的值的参照完整性；

约束用于预防破坏表之间连接的行为；

约束也能防止非法数据插入外键列，因为它必须是它指向的那个表中的值之一；

例子：

```

CREATE TABLE Orders
(
O_Id int NOT NULL,
OrderNo int NOT NULL,
P_Id int,
PRIMARY KEY (O_Id),
FOREIGN KEY (P_Id) REFERENCES Persons(P_Id)
)

//如需命名 FOREIGN KEY 约束，并定义多个列的 FOREIGN KEY 约束，请使用下面的 SQL 语法：
CREATE TABLE Orders
(
O_Id int NOT NULL,
OrderNo int NOT NULL,
P_Id int,
PRIMARY KEY (O_Id),
CONSTRAINT fk_PerOrders FOREIGN KEY (P_Id)
REFERENCES Persons(P_Id)
)

//当 "Orders" 表已被创建时，如需在 "P_Id" 列创建 FOREIGN KEY 约束
ALTER TABLE Orders
ADD FOREIGN KEY (P_Id)
REFERENCES Persons(P_Id)

//撤销 FOREIGN KEY 约束
ALTER TABLE Orders
DROP FOREIGN KEY fk_PerOrders

```

### CHECK :检查约束 

如果对单个列定义 CHECK 约束，那么该列只允许特定的值。 保证列中的值符合指定的条件。
  
如果对一个表定义 CHECK 约束，那么此约束会基于行中其他列的值在特定的列中对值进行限制。

例子：

```

下面的 SQL 在 "Persons" 表创建时在 "P_Id" 列上创建 CHECK 约束。CHECK 约束规定 "P_Id" 列必须只包含大于 0 的整数。
CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
CHECK (P_Id>0)
)

//添加CHECK约束
ALTER TABLE Persons
ADD CHECK (P_Id>0)

//删除约束
ALTER TABLE Persons
DROP CHECK chk_Person

```

### DEFAULT - 规定没有给列赋值时的默认值。

例子：

```

CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255) DEFAULT 'Sandnes'
)

//创建DEFAULT约束
ALTER TABLE Persons
ALTER City SET DEFAULT 'SANDNES'

//如需撤销 DEFAULT 约束
ALTER TABLE Persons
ALTER City DROP DEFAULT

注意：
每个表可以有多个 UNIQUE 约束，但是每个表只能有一个 PRIMARY KEY 约束。

```

### AUTO INCREMENT

```
该字段会使得每次新纪录插入表中时，生成唯一的数字。
可以通过 alter table persons auto_increment=100; 将自增值置为需要的值。

例子：
mysql> create table persons(ID int not null auto_increment,lastName varchar(255) not null ,city varchar(255), PRIMARY KEY(ID));
Query OK, 0 rows affected (0.29 sec)

mysql> insert into persons(lastName,city) values('zhangsan','beijing');
Query OK, 1 row affected (0.04 sec)

mysql> select * from persons;
+----+----------+---------+
| ID | lastName | city    |
+----+----------+---------+
|  1 | zhangsan | beijing |
+----+----------+---------+
1 row in set (0.00 sec)

mysql> alter table persons auth_increment=100;


mysql> insert into persons(lastName,city) values('lisi','handan');
Query OK, 1 row affected (0.05 sec)

mysql> select * from persons;
+-----+----------+---------+
| ID  | lastName | city    |
+-----+----------+---------+
|   1 | zhangsan | beijing |
| 100 | lisi     | handan  |
+-----+----------+---------+
2 rows in set (0.01 sec)

```

## 级联约束 

有了合适的外键约束后，如果读者视图插入新行或者修改行而导致父表中的外键列并无匹配值，则服务器会抛出异常

例子：

```

mysql> select * from product;
+------------+-------------------------+-----------------+--------------+--------------+
| product_cd | name                    | product_type_cd | date_offered | date_retired |
+------------+-------------------------+-----------------+--------------+--------------+
| AUT        | auto loan               | LOAN            | 2000-01-01   | NULL         |
| BUS        | business line of credit | LOAN            | 2000-01-01   | NULL         |
| CD         | certificate of deposit  | ACCOUNT         | 2000-01-01   | NULL         |
| CHK        | checking account        | ACCOUNT         | 2000-01-01   | NULL         |
| MM         | money market account    | ACCOUNT         | 2000-01-01   | NULL         |
| MRT        | home mortgage           | LOAN            | 2000-01-01   | NULL         |
| SAV        | savings account         | ACCOUNT         | 2000-01-01   | NULL         |
| SBL        | small business loan     | LOAN            | 2000-01-01   | NULL         |
+------------+-------------------------+-----------------+--------------+--------------+
8 rows in set (0.00 sec)

mysql> select * from product_type;
+-----------------+-------------------------------+
| product_type_cd | name                          |
+-----------------+-------------------------------+
| ACCOUNT         | Customer Accounts             |
| INSURANCE       | Insurance Offerings           |
| LOAN            | Individual and Business Loans |
+-----------------+-------------------------------+
3 rows in set (0.01 sec)

mysql> update product set product_type_cd ='xyz' where product_type_cd ='LOAN';
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`bank`.`product`, CONSTRAINT `fk_product_type_cd` FOREIGN KEY (`product_type_cd`) REFERENCES `product_type` (`product_type_cd`))


```

在product.product_type_cd列上有外键约束，而product_type表中没有哪一行的product_type_cd的列值为xyz，所有不会更新成功。
所谓级联的更新，在删除存在的外键和添加新的外键时包含on update cascade，这些外键约束的变化能够实现传播。

需要做以下修改

```

alter table product add constraint fk_product_type_cd foreign key (product_type_cd) references product_type(product_type_cd) on update cascade;

```

# SQL索引 INDEX

CREATE INDEX 语句用于在表中创建索引，在不读取整个表的情况下，索引数据库应用程序可以更快的查找数据。

更新一个包含索引的表需要比更新一个没有索引的表花费更多的时间，这是由于索引本身也需要更新。

索引并不包含实体中的所有数据，而是那些用于定位表中的数据的列，以及描述行信息的列。

innodb引擎是支持事务的行锁引擎；mylsam引擎是不支持事务的表锁引擎，它支持全文索引（文本索引）

一般的索引为b树索引（平衡树索引） 它有一个或者多个分之节点，分之节点又指向单级叶子节点。分支节点用于遍历树，叶节点则保存真正的值和位置信息 。


## 显示查询执行计划 （待完善）

```

mysql> explain select * from account where cust_id in(1,5,9,11);
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | account | NULL       | ALL  | fk_a_cust_id  | NULL | NULL    | NULL |   24 |    33.33 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+


```

## 单列索引

 下面的 SQL 语句在 "Persons" 表的 "LastName" 列上创建一个名为 "PIndex" 的索引：

```
  CREATE INDEX PIndex ON Persons(LastName);

```

## 唯一索引

除了常规的索引的用途外，还可以限制索引列出现重复值。

```

mysql> alter table department add unique dept_name_unique_idx(name);
Query OK, 0 rows affected (0.43 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show index from department;
+------------+------------+----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table      | Non_unique | Key_name             | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+------------+------------+----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| department |          0 | PRIMARY              |            1 | dept_id     | A         |           2 |     NULL | NULL   |      | BTREE      |         |               |
| department |          0 | dept_name_unique_idx |            1 | name        | A         |           2 |     NULL | NULL   |      | BTREE      |         |               |
| department |          1 | dept_name_idx        |            1 | name        | A         |           2 |     NULL | NULL   |      | BTREE      |         |               |
+------------+------------+----------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
3 rows in set (0.00 sec)


```

## 多列索引

 如果您希望索引不止一个列，您可以在括号中列出这些列的名称，用逗号隔开：

```

  CREATE INDEX PIndex ON Persons(LastName,FirstName);
或者

mysql> alter table employee add index emp_name_idx(lname,fname);
Query OK, 0 rows affected (0.37 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show index from employee;
+----------+------------+----------------+--------------+--------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table    | Non_unique | Key_name       | Seq_in_index | Column_name        | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+----------+------------+----------------+--------------+--------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| employee |          0 | PRIMARY        |            1 | emp_id             | A         |          18 |     NULL | NULL   |      | BTREE      |         |               |
| employee |          1 | fk_e_emp_id    |            1 | superior_emp_id    | A         |           7 |     NULL | NULL   | YES  | BTREE      |         |               |
| employee |          1 | fk_dept_id     |            1 | dept_id            | A         |           3 |     NULL | NULL   | YES  | BTREE      |         |               |
| employee |          1 | fk_e_branch_id |            1 | assigned_branch_id | A         |           4 |     NULL | NULL   | YES  | BTREE      |         |               |
| employee |          1 | emp_name_idx   |            1 | lname              | A         |          18 |     NULL | NULL   |      | BTREE      |         |               |
| employee |          1 | emp_name_idx   |            2 | fname              | A         |          18 |     NULL | NULL   |      | BTREE      |         |               |
+----------+------------+----------------+--------------+--------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
6 rows in set (0.00 sec)


```

在创建多列索引时，必须仔细考虑那一列作为第一列，那一列作为第二列。

## 添加索引

```
mysql> alter table department add index dept_name_idx(name);
Query OK, 0 rows affected (0.35 sec)
Records: 0  Duplicates: 0  Warnings: 0

```

## 删除索引

```

mysql> alter table department drop index dept_name_idx;
Query OK, 0 rows affected (0.20 sec)
Records: 0  Duplicates: 0  Warnings: 0

```

## 查看索引

```

mysql> show index from department;
+------------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table      | Non_unique | Key_name      | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+------------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| department |          0 | PRIMARY       |            1 | dept_id     | A         |           2 |     NULL | NULL   |      | BTREE      |         |               |
| department |          1 | dept_name_idx |            1 | name        | A         |           2 |     NULL | NULL   |      | BTREE      |         |               |
+------------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
2 rows in set (0.00 sec)


```

## 索引的使用

每一个索引实际上都是一个特殊的表，每次在添加或者删除行时，表中的所有索引都被修改。当数据越来越多时，会拖慢服务器的处理速度

仅仅当出现清晰需求时才添加索引。

如有特殊需求需要索引，可以添加索引，运行程序，然后删除索引。下次也是如此。

**默认策略**
1. 确保所有的逐渐被索引，大部分数据库服务器在创建主键约束的时候会自动生成唯一索引；
2. 为所有被外键约束引用的列创建索引。在服务器准备删除父行时会搜索对应的子行是否存在，它必须发出一个查询搜索列中的特殊值。
3. 大多是日期可以作为索引；


# CREATE VIEW 视图

视图是基于 SQL 语句的结果集的可视化的表，不同于表的是，视图不涉及数据存储，因此用户不必担心数据会充满磁盘空间、

视图包含行和列，就像一个真实的表。视图中的字段就是来自一个或多个数据库中的真实的表中的字段。

您可以向视图添加 SQL 函数、WHERE 以及 JOIN 语句，也可以呈现数据，就像这些数据来自于某个单一的表一样。


## 视图的使用情况

1. 可以保证数据的安全性；
2. 数据聚合；
3. 隐藏复杂性；
4. 连接分区数据；




## 创建视图

```
CREATE VIEW view_name AS
SELECT column_name(s)
FROM table_name
WHERE condition

```

视图总是显示最新的数据！每当用户查询视图时，数据库引擎通过使用视图的 SQL 语句重建数据。

## 查看视图

```

SELECT * from view_name;

```

## 更新视图

满足以下条件，视图则可以被更新。

1. 没有使用聚合函数；
2. 没有使用having 或者group by语句；
3. 没有使用union all 、union、distinct语句；
4. from包括不止一个表或者视图；


```

CREATE OR REPLACE VIEW view_name AS
SELECT column_name(s)
FROM table_name
WHERE condition

```

## 撤销视图

```
DROP VIEW view_name;

```

例子：

```

create view viewTest as select * from persons where ID>10;

mysql> select * from viewTest;
+-----+----------+--------+
| ID  | lastName | city   |
+-----+----------+--------+
| 100 | lisi     | handan |
+-----+----------+--------+
1 row in set (0.00 sec)

mysql> drop view viewTest;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from viewTest;
ERROR 1146 (42S02): Table 'testDB.viewTest' doesn't exist


```


# 事务

1. 事务是指，一堆sql要么执行，要么执行，这成为原子性。

2. 说到事务，就必须提锁。锁时数据库服务器用来控制数据资源被并行使用的一种机制。

3. 数据库的写操作必须向服务器申请写锁才能够修改数据。而读操作必须获得读锁才能读取数据。

4. 读写锁是指，一次事务只能有一个写锁不能有读锁或者多个读锁不能有写锁。

5. 服务器要保证从查询开始到查询结束看到一个一致性的数据视图。这个方法成为版本控制。

6. 锁的粒度分为表锁、页锁、行锁。

7. 表锁需要较少的簿记就可以锁定整个表，但是用户增多时他会迅速产生不可接受的等待时间；行锁需要更多的簿记，但是各个用户在不同的行，允许多个用户同时修改一个表。

8. 如果不显式的启动一个事务，单个sql会被独立于其它语句自动提交。启动事务必须先提交一个命令。

9. mysql服务器时默认的自动提交事务，但是可以修改。

10. rollback事务回滚，commit提交事务

关于意外情况
1. 服务器宕机，服务器重启时事务会自动回滚；
2. 提交一个SQL模式语句，比如alter table会引起当前事务提交和新事务的启动；
3. 提交一个start transcation命令，将会引起一个新事务提交；
4. 因为服务器检测到一个思索并且确定当前事务就是罪魁祸首，则服务器会结束当前事务，然后事务将会被回滚，同时释放错误信息。


# 元数据

### 元数据的类型

1. 表名
2. 表存储信息（表空间、初始值大小）
3. 存储引擎
4. 列名
5. 列数据类型
6. 默认值
7. 非空列约束
8. 主键列
9. 主键名
10. 主键索引名
11. 索引名
12. 索引类型
13. 索引列
14. 索引列排序顺序
15. 索引存储信息
16. 外键名
17. 外键列
18. 外键的关联表

### 视图的信息模式

```

mysql> select * from information_schema.tables where table_schema='bank' order by 1;
+---------------+--------------+-----------------+------------+--------+---------+------------+------------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+-------------------+----------+----------------+---------------+
| TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME      | TABLE_TYPE | ENGINE | VERSION | ROW_FORMAT | TABLE_ROWS | AVG_ROW_LENGTH | DATA_LENGTH | MAX_DATA_LENGTH | INDEX_LENGTH | DATA_FREE | AUTO_INCREMENT | CREATE_TIME         | UPDATE_TIME         | CHECK_TIME | TABLE_COLLATION   | CHECKSUM | CREATE_OPTIONS | TABLE_COMMENT |
+---------------+--------------+-----------------+------------+--------+---------+------------+------------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+-------------------+----------+----------------+---------------+
| def           | bank         | customer_viewer | VIEW       | NULL   |    NULL | NULL       |       NULL |           NULL |        NULL |            NULL |         NULL |      NULL |           NULL | NULL                | NULL                | NULL       | NULL              |     NULL | NULL           | VIEW          |
| def           | bank         | customer        | BASE TABLE | InnoDB |      10 | Dynamic    |         11 |           1489 |       16384 |               0 |            0 |         0 |             14 | 2017-11-11 23:46:08 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | business        | BASE TABLE | InnoDB |      10 | Dynamic    |          4 |           4096 |       16384 |               0 |            0 |         0 |           NULL | 2017-11-11 23:46:08 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | transaction     | BASE TABLE | InnoDB |      10 | Dynamic    |         21 |            780 |       16384 |               0 |        49152 |         0 |             22 | 2017-11-11 23:46:10 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | branch          | BASE TABLE | InnoDB |      10 | Dynamic    |          4 |           4096 |       16384 |               0 |            0 |         0 |              5 | 2017-11-11 23:46:06 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | product_type    | BASE TABLE | InnoDB |      10 | Dynamic    |          3 |           5461 |       16384 |               0 |            0 |         0 |           NULL | 2017-11-11 23:46:07 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | account         | BASE TABLE | InnoDB |      10 | Dynamic    |         24 |            682 |       16384 |               0 |        65536 |         0 |             30 | 2017-11-11 23:46:09 | 2017-12-01 00:09:25 | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | product         | BASE TABLE | InnoDB |      10 | Dynamic    |          8 |           2048 |       16384 |               0 |        16384 |         0 |           NULL | 2017-12-03 02:17:13 | 2017-12-03 02:17:13 | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | officer         | BASE TABLE | InnoDB |      10 | Dynamic    |          4 |           4096 |       16384 |               0 |        16384 |         0 |              5 | 2017-11-11 23:46:09 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | individual      | BASE TABLE | InnoDB |      10 | Dynamic    |          9 |           1820 |       16384 |               0 |            0 |         0 |           NULL | 2017-11-11 23:46:08 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | employee_vw     | VIEW       | NULL   |    NULL | NULL       |       NULL |           NULL |        NULL |            NULL |         NULL |      NULL |           NULL | NULL                | NULL                | NULL       | NULL              |     NULL | NULL           | VIEW          |
| def           | bank         | employee        | BASE TABLE | InnoDB |      10 | Dynamic    |         18 |            910 |       16384 |               0 |        49152 |         0 |             19 | 2017-12-03 01:19:37 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
| def           | bank         | department      | BASE TABLE | InnoDB |      10 | Dynamic    |          2 |           8192 |       16384 |               0 |            0 |         0 |              4 | 2017-12-03 01:17:04 | NULL                | NULL       | latin1_swedish_ci |     NULL |                |               |
+---------------+--------------+-----------------+------------+--------+---------+------------+------------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+-------------------+----------+----------------+---------------+
13 rows in set (0.00 sec)


```

### information_schema数据库表说明:


　　information_schema数据库是MySQL自带的，它提供了访问数据库元数据的方式。什么是元数据呢？元数据是关于数据的数据，如数据库名或表名，列的数据类型，或访问权限等。有些时候用于表述该信息的其他术语包括“数据词典”和“系统目录”。
　　在MySQL中，把 information_schema 看作是一个数据库，确切说是信息数据库。其中保存着关于MySQL服务器所维护的所有其他数据库的信息。如数据库名，数据库的表，表栏的数据类型与访问权 限等。在INFORMATION_SCHEMA中，有数个只读表。它们实际上是视图，而不是基本表，因此，你将无法看到与之相关的任何文件。


**SCHEMATA表**：提供了当前mysql实例中所有数据库的信息。是show databases的结果取之此表。

**TABLES表**：提供了关于数据库中的表的信息（包括视图）。详细表述了某个表属于哪个schema，表类型，表引擎，创建时间等信息。是show tables from schemaname的结果取之此表。

**COLUMNS表**：提供了表中的列信息。详细表述了某张表的所有列以及每个列的信息。是show columns from schemaname.tablename的结果取之此表。

**STATISTICS表**：提供了关于表索引的信息。是show index from schemaname.tablename的结果取之此表。

**USER_PRIVILEGES**（用户权限）表：给出了关于全程权限的信息。该信息源自mysql.user授权表。是非标准表。

**SCHEMA_PRIVILEGES**（方案权限）表：给出了关于方案（数据库）权限的信息。该信息来自mysql.db授权表。是非标准表。

**TABLE_PRIVILEGES**（表权限）表：给出了关于表权限的信息。该信息源自mysql.tables_priv授权表。是非标准表。

**COLUMN_PRIVILEGES**（列权限）表：给出了关于列权限的信息。该信息源自mysql.columns_priv授权表。是非标准表。

**CHARACTER_SETS**（字符集）表：提供了mysql实例可用字符集的信息。是SHOW CHARACTER SET结果集取之此表。

**COLLATIONS表**：提供了关于各字符集的对照信息。

**COLLATION_CHARACTER_SET_APPLICABILITY**表：指明了可用于校对的字符集。这些列等效于SHOW COLLATION的前两个显示字段。

**TABLE_CONSTRAINTS**表：描述了存在约束的表。以及表的约束类型。

**KEY_COLUMN_USAGE**表：描述了具有约束的键列。

**ROUTINES**表：提供了关于存储子程序（存储程序和函数）的信息。此时，ROUTINES表不包含自定义函数（UDF）。名为“mysql.proc name”的列指明了对应于INFORMATION_SCHEMA.ROUTINES表的mysql.proc表列。

**VIEWS**表：给出了关于数据库中的视图的信息。需要有show views权限，否则无法查看视图信息。

**TRIGGERS**表：提供了关于触发程序的信息。必须有super权限才能查看该表于触发程序的信息。必须有super权限才能查看该表


 
















 


  


 