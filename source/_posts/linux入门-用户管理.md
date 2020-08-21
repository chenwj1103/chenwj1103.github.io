---
title: linux入门-用户管理
copyright: true
date: 2018-02-10 00:40:10
tags: linux入门
categories: linux

---

# linux用户和用户组

linux是一个多用户分时系统,想要使用系统必须使用账号,而且账号必须设置密码.各个用户可以设置不同的文件权限,保证不同用户的数据安全.各个账号都有对应的用户id和用户组id.

## UID和GID

为了区分不同的用户,我们可以根据用户名去区分,但是对于OS,它其实是没有意义的字符串.linux采用32位的整数来记录用户id,简称UID,计算机会记录用户id和用户名称的关系.linux用户分为三种:普通用户,根用户和系统用户.

- 普通用户

普通用户是指使用linux系统的真是用户,这类用户的用户名/密码/权限都有详细的设置.普通用户只能在家目录/系统临时目录或其它授权的目录下操作.普通用户的id从500开始.

- 根用户

根用户也是root用户,它的id为0,也就是超级用户.root用户拥有对系统的完全控制权.

- 系统用户

系统用户是指系统运行时必须要有的用户,但并不是真实的使用者.id范围为1至500


## 查看当前用户的groupId和userId 

````
zhuningning@ubuntu:~$ id
uid=1000(zhuningning) gid=1000(zhuningning) 组=1000(zhuningning),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
````
## 查看用户所属的用户组

````
zhuningning@ubuntu:~$ groups
zhuningning adm cdrom sudo dip plugdev lpadmin sambashare

````

## 查看当前登录的用户

````
zhuningning@ubuntu:~$ who
zhuningning tty7         2018-05-31 02:04 (:0)
zhuningning@ubuntu:~$ w
 23:23:19 up 21:19,  1 user,  load average: 0.47, 0.52, 0.31
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
zhuningn tty7     :0               02:04   21:19m  2:18   0.20s /sbin/upstart --user
zhuningning@ubuntu:~$ users
zhuningning

````
## 记录系统的用户名和密码的信息的文件
````
cat /etc/passwd
cat /etc/shadow

````

## 查看隐藏的文件 

````
zhuningning@ubuntu:/home$ ls -la
总用量 20
drwxr-xr-x  5 root        root        4096 Sep 10  2017 .
drwxr-xr-x 24 root        root        4096 May  9 07:06 ..
drwxr-xr-x  2        1003        1003 4096 Sep 10  2017 chenweijie
drwxr-xr-x  2 chenwj      chenwj      4096 Sep 10  2017 chenwj
drwxr-xr-x 51 zhuningning zhuningning 4096 May 31 02:04 zhuningning

````

## 新建用户

1. adduser john 添加用户
2. useradd -u 555 user1   为用户user1指定uId ,当然该id必须是唯一的
3. useradd -g user1 user2   为用户user2指定用户组为user1 
4. useradd -d /home/mydir3 user3  为user3指定家目录

## 修改密码
 
用户创建后没有密码是不可以登录系统的，只有设置了密码才可以登录系统。 

````
zhuningning@ubuntu:/home$ sudo passwd chenweijie
输入新的 UNIX 密码： 

````

## 关于添加用户以及授权的操作

1.adduser  chenweijie 添加用户

2.授权 修改 /etc/sudoers 文件，找到下面一行，在root下面添加一行，如下所示：
````
Allow root to run any commands anywhere
root    ALL=(ALL)     ALL
chenweijie   ALL=(ALL)     ALL
````

3.语法:
````
     useradd 选项 用户名
语义:
  -c comment            指定一段注释性描述。
  -d 目录                   指定用户主目录，如果此目录不存在，则同时使用-m选项，可以创建主目录。
  -g 用户组               指定用户所属的用户组。
  -G 用户组 用户组   指定用户所属的附加组。
  -s Shell文件            指定用户的登录Shell。
  -u 用户号               指定用户的用户号，如果同时有-o选项，则可以重复使用其他用户的标识号。
  用户名                   指定新用户的登录名。
userdel 选项 用户名

选项:
  -r,  把用户的主目录一起删除。
 usermod 选项 用户名
选项:
   包括-c, -d, -m, -g, -G, -s, -u以及-o等,
   这些选项的意义与useradd命令中的选项一样，可以为用户指定新的资源值。另外，有些系统可以使用如下选项：
   -l 新用户名  指定一个新的账号，即将原来的用户名改为新的用户名。

usermod -G groupname username 给已有的用户增加工作组
newgrp   groupName 切换到用户组 以获取该组的权限
groups 查看该当前用户所属的用户组，第一个是主要用户组。

````
## 关于用户权限的操作

添加组的命令： groupadd 组名 。 （在root管理权限）

查看linux中所有组的信息： cat /etc/group 。

创建用户，并同时指定将该用户分配到哪个组里： useradd -g 组名 用户名。 （在root管理权限）

查看linux中所有用户的信息： cat /etc/passwd 。

更改某个用户所在的组： usermod -g 组名 用户名。 （在root管理权限）

## 用户的切换

````
exit 退出当前用户。

su - 切换到用root用户是，不但身份变成了root ，而且还可以是用root的用户环境。

sudo 是在sudo后加上要使用的命令，但是需要为该用户配置 /etc/sudoers 中的权限 
    	root	ALL=(ALL:ALL) ALL 改命令表示该用户可以在任何地方登录后执行任何人的任何命令。
    	但是每次需要输入密码，如果想要不输入密码，则可以在最后设置为 NOPASSWD :ALL
su是切换用户，su -是切换用户并且使用用户的环境，而sudo并没有切换用户，而是使用用户的身份和权限执行了命令。

````

## 例行任务管理

1.单一时刻执行一次任务  at      atrm

````
at  now + 20 minutes 
/sbin/shutdown -h now
执行组合键 ctrl+D

````

2.atq

查看任务队列

````
zhuningning@ubuntu:~$ atq
4	Fri Jun  1 01:05:00 2018 a zhuningning

````

3.atrm taskNum

删除任务

````
zhuningning@ubuntu:~$ atrm 4
zhuningning@ubuntu:~$ atq
zhuningning@ubuntu:~$

````
 
4.周期性的执行任务  

````
先启动:service crond start 

编辑：crontab -e
 
查看任务:crontab -l 

删除所有的任务 : crontab -r 

````


