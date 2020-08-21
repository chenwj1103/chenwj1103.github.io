---
title: linux入门-文件管理
copyright: true
date: 2018-06-01 00:52:17
tags: linux文件管理
categories: linux

---

# 文件和目录管理

1.FHS(文件系统层次标准)

````
bin 常见的用户指令
/boot 内核和启动文件
/dev 设备文件
/etc 系统和服务的配置文件
/home 系统默认的普通用户的家目录
/lib 系统的函数库目录
/lost+found ext3文件系统需要的目录，用于磁盘检查
/mnt 系统加载文件系统时常用的挂载点
/opt 第三方软件安装目录
/proc 虚拟文件系统
/root root用户的家目录
/sbin 存放系统管理命令
/tmp 临时文件的存放目录
/usr 存放与用户直接相关的文件和目录
/media 系统用来挂载光驱等临时文件系统的挂载点

````
2.绝对路径

  绝对路径是以‘/’开头的路径
   	
  pwd 查看当前文件所在的路径
  
3.特殊目录

每个目录下都有两个文件 （.）(..) 所有以.开始的文件都是隐藏文件 

./代表当前目录 ../代表上层目录

使用ls -la可以看到隐藏文件

4.相对路径

   cd ..进入上层目录   cd / 进入根目录
   
5.文件的操作

1）创建文件 touch test.txt  如果创建时该文件已经存在，则不会改变文件的内容，会改变文件的创建时间
 
2）删除文件 rm  test.txt 然后会提示 输入y 确定删除，输入n 反悔

3）移动文件 mv test.txt  /home/chen  后面带有两个参数 第一个时要被移动的文件 第二个参数要移动到的目录； 如果第二个参数是文件名字 则表示重命名操作

4）查看文件 cat  test.txt    如果带一个参数cat -n test.txt  则会显示文件行号

5）查看文件的头部 head -n 100 test.txt  显示文件的前100行

6）查看文件的尾部 tail -n 100 test.txt   更重要的可以使用 tail -f test.txt 动态的追踪文件的尾部 

7）dos格式的文本转换成unix格式的文本 dos2unix


6.目录的相关操作

1) 进入目录 cd    change directory的缩写

2) 创建目录 mkdir  test/      可以使用-p 参数一次创建一个目录和其子目录  mkdir  -p  dir1/dir2

3) 删除目录 rm  如果目录不为空，则需要先删除目录中的文件。或者可以使用rm -r 删除 ，而rm -rf，则时强制无提示的删除目录以及目录下的文件

4)复制文件或者目录 cp 第一个参数是要复制的源文件，第二个是要复制到的目录或者复制后的文件名字。

cp test.txt  test2.txt (改名字) 
   
cp  test.txt  /temp/ (复制到temp目录下不改名字)  
  
cp  test.txt  /temp/test2.txt (复制到temp目录下改名字)
    
复制目录则需要加 -r      cp  -r dir1  dir2

5)文件的时间戳 使用touch命令可以修改文件的时间戳


7.权限含义详解：

````
drwxrwxr-x  2 zhuningning zhuningning 4096 Sep 11 23:21 dir1/

权限分为三种： r 代表可读，用4表示。 w 代表可写，用2表示。 x 代表可执行，用1表示。 
-rw-r--r--  的解释如下, ：
第一位的‘-’ 代表文件类型。 【'd' 代表目录，'|' 代表链接 ，‘-’代表普通文件】。
第二位到第四位的‘rw-’，代表该文件的所有者对该文件的权限。
第五位到第七位的‘r--’，代表该文件所在的组的其他用户对该文件的权限。
第八位到第十位的‘r--’，代表其他组的用户对该文件的权限。

 
chmod命令：使用例： chmod 760 目录/文件，
意思是 给该目录/文件的所有者对该目录/文件赋予rwx权限，给该目录/文件所在的组的其他用户对该目录/文件赋予rw-权限，给其他组的用户对该目录/文件赋予---权限。
 
chmod 755 abc ：赋予abc权限rwxr-xr-x。
chmod u=rwx,g=rx,o=rx abc ：同上 u=用户权限  g=组权限 o=不同组其他用户权限。
chmod u-x,g+w abc ：给abc去除用户执行的权限，增加组写的权限。-表示去除  +表示增加
chmod a+r abc ：给所有用户添加读的权限。
chmod -R 754 abc ：给abd目录以及目录下的文件赋予权限754
 
改变拥有者（chown）和用户组（chgrp）命令：
chown xiaoming abc ：改变abc的拥有者为xiaoming。
chgrp root abc ：改变abc所属的组为root。
chown root ./abc ： 改变abc目录的所有者是root。
chown -R root ./abc ：改变abc目录及其下面的所有目录和文件的所有者是root。参数-R 是递归的意思。

文件的隐藏属性：lsattr test.txt     chattr +a test.txt   表示文件不可以被删除，即使root权限也不可以；  chattr +i test.txt   表示文件不可以被删除/修改，即使root权限也不可以
suid权限 （u权限）普通用户可以使用root的身份来执行命令 chmod u+s  files 来授权
sgid 权限（g权限） 组可以使用root身份来执行名  chmod g+s 来授权
sticky权限（s权限），只能使用在目录上，设置了这种权限，任何用户可以创建和修改文件，但是只有文件的创建者和root用户可以删除自己的文件 chmod o+t  dir 

````

8.系统创建文件和目录的默认权限

umask（遮罩值） 对于uid大于 99的 umask为022 否则为002 

文件的默认权限是 666 目录的权限是777 

root用户的文件权限是644 目录权限是 755

普通用户的文件权限是664 目录是775

````

给用户组添加SGID权限:
chmod g+s somefile

给用户添加SUID权限
chmod u+s somefile
````
Sticky权限还能用于设置在目录上,设置了这种权限的目录,任何用户都可以在该目录创建或者修改文件,但是该文件的创建者和root用户可以删除这个文件


9.查看文件类型（可以显示文件的特殊属性）
````
file  /tmp/
该目录是一个拥有sticky属性的目录  /tmp/: sticky, directory

file  /etc/passwd
该文件是一个ASCII编码的文本文件  /etc/passwd: ASCII text

````
10. 查找文件

1)一般查找 find PATH -name FILENAME   //在某个路径下查找文件

2）locate fileName  //该命令依赖于一个数据库，一般执行命令之前需要执行updatedb命令来更新数据库

3）which/whereis  fileName  //该命令从系统的path变量所定义的目录中找到可执行文件的绝对路径 ，而whereis除此之外还可以还能找到其相关的二进制文件

11.压缩文件

````
gzip和gunzip 用来压缩和解压单个文件

tar不仅可以打包文件，还可以讲整个目录中的全部文件整合成一个包，整合包的同时还可以使用gzip进行压缩。可以使用.tar.gz作为后缀简写为.tgz

tar -zcvf boot.tgz /boot 

-z代表使用gzip进行压缩，-c表示创建压缩文件 -v表示是显示当前被压缩的文件，-f代表使用文件名字 也就是boot.tgz

tar -zxvf  boot.tgz -C /tmp  
 -x表示解压的意思  -C 执行解压到/tmp命令
 
bzip2 fileName  进行压缩
-z参数进行强制压缩   -d进行强制解压

````

12 .查看文件的大小

````
ll -h test.txt
-rwxrwxrwx 1 zhuningning zhuningning 152M Feb  7  2017 crossover_16.1.0-1.deb*
````
