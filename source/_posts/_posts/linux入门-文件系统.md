---
title: linux入门-文件系统
copyright: true
date: 2018-06-08 00:55:19
tags: linux,文件系统
categories: linux

---

# 文件系统

1.文件系统其实就是组织文件的方法。
linux支持多种文件系统，最常用的就是ext2  /ext3
linux系统都具有通用的结构，包括：超级快 i节点 数据块 目录块四个部分。
ext3相对于ext2的优势时其支持日志功能。
磁盘分区分为主分区和扩展分区，一块磁盘最多可以扩展4个主分区。
磁盘分区完成后需要进行创建文件系统的操作，最后讲该分区挂载到系统中的某个挂载点才可以使用。

2.   fdisk 创建文件系统 
mount 挂载磁盘   挂载点只能是目录  （暂时挂载）
 /etc/sftab 设置启动自动挂载 

3.磁盘检查
当磁盘出现逻辑错误时，可以使用fsck来尝试修复。（但是此时磁盘必须时未挂载的状态，可以使用umount命令进行卸载）
umount  /DEVICE/PATH 卸载挂载盘（参数可以时设备路径或者挂载点）
使用fsck -t TYPE  /DEVICE/PATH   TYPE可以是ext2 ext3 最后时设备的全路径

4.linux的逻辑卷
逻辑卷是使用逻辑卷组管理（LVM）创建出来的设备，LVM时介于硬盘裸设备和文件系统的中间层。
物理卷（PV）   卷组（VG）时物理组的集合，逻辑卷(LV)是从PV中划出来的一块逻辑磁盘。
5.创建物理卷：（pvcreate pvdisplay）
首先提供一个磁盘，然后创建三个分区 fdisk	/dev/sdc
fdisk -l 查看是否创建成功
修改各个分区的id为8e
创建PV  pvcreate /dev/sdc1
pvscan 查看系统中的PV，pvdisplay可以显示更详细的信息。

6.创建并查询卷组（vgcreate  vgdisplay）
创建卷组：vgcreate VG_NAME DEVICE1   DEVICE2  

7.扩容卷组（vgextend）
扩展卷组：vgextend VG_NAME DEVICE1 DEVICE2

8.创建逻辑卷（lvcreate lvdisplay）
有了卷组就可以创建逻辑卷 lvcreate  -L SIZE -n LV_NAME VG_NAME    -L制定逻辑卷的大小size表示具体逻辑卷的大小 -n表示逻辑卷的名称

9. 创建文件系统并挂载
逻辑卷和使用福利分区一样，需要先创建文件系统  挂载后才能被系统使用。
创建文件系统    mkfs.ext3 /dev/first_VG/first_LV 
创建一个挂载点  mkdir /root/newLV
挂载 mount  /dev/first_VG/first_LV/root/newLV

10.硬链接
在linux系统中，所有的文件都会有一个编号，成为索引节点。多个文件名指向同一索引节点，这种链接称为硬链接。
删除一个链接不会影响索引节点本身和其他链接，只有最后一个链接被删除时，文件的数据块以及目录的链接才会被释放。
硬链接有两个限制：
不允许给目录创建硬链接；
只有在同一个文件系统的文件才可以创建链接。

ls  -li显示文件 
4849841 drwxr-xr-x  2 zhuningning zhuningning      4096 Sep 11 20:32 ./
第一列表示索引节点
 第三列表示源文件的关联数，
只有此数为0的时候才会真正的被文件系统删除

创建方法： ln  test.txt  test_ln    为test.txt 文件创建链接

11.软链接
是指一个包含了另一个文件路径名的文件，可以指向任意文件或者目录，他也不可以跨文件系统。
如果删除了源文件，软链接则会断链。(在linux下，会使用特殊标记表示该文件时断链。)
源文件和软链接文件的inode时不一样的，代表他们时2个不一样的文件。
创建软链接 ln -s  test.txt  test_ln.txt


# 字符处理

1.管道：它时一个固定大小的缓冲区，该缓冲区的大小为一字节。
2.它可以把一个命令的输入内容当做下一个命令的输入内容，两个命令之间只需要使用管道符  “|” 连接即可。
3. 比如：
ls -l  /etc/init.d/ |more (第一个命令的输入作为第二个命令的输入)
4.grep是基于行的文本搜索工具， 
参数：
-i 不区分大小写
-c 统计包含匹配的行数
-n 输出行号
-v反向匹配

5.sort 对输出内容直接排序 
参数
-n 采用数字排序
-t 指定分隔符
-k 指定第几行
-r 反向排序

cat  sort.txt  | sort 
默认按照每行的第一个字母进行排序
a:4
b:3
c:2
d:1
e:5
f:11

cat sort.txt |sort -t ":" -k 2
d:1
f:11
c:2
b:3
a:4
e:5

6.uniq 删除连续的完全一致的行
参数：
-i 忽略大小写
-c 计算重复行数
cat uniq.txt |uniq
abc
123
abc
123
cat uniq.txt |sort|uniq -c
 2 123
      	2 abc

7.使用cut截取文本
cut  -f 指定的行  -d '分隔符'
cat /etc/passwd |cut -f1 -d ':'

cut -c 指定列的字符
cat /etc/passwd |cut -c1-5,7-10

8 split 分割文件

9 paste合并文件


10.split
