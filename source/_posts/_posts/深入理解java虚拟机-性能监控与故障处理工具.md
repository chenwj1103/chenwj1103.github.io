---
title: 深入理解java虚拟机-性能监控与故障处理工具
copyright: true
date: 2018-10-28 15:10:03
tags: 性能监控与故障处理工具
categories: jvm

---

jdk的性能监控和故障处理工具如下图：

![JDK的故障处理工具](/images/jvm/JDK的故障处理工具.png)

[jstat工具](http://www.51testing.com/html/92/77492-203728.html)

[jconsole工具使用](https://www.cnblogs.com/baihuitestsoftware/articles/6405580.html)

[jvisualvm 工具使用](https://www.cnblogs.com/kongzhongqijing/articles/3625340.html)

[jstack工具使用](https://www.cnblogs.com/kongzhongqijing/articles/3630264.html)

# jps：虚拟机进程状况工具

## JPS 名称: 

jps - Java Virtual Machine Process Status Tool

## 命令用法
````
: jps [options] [hostid]
              options:命令选项，用来对输出格式进行控制
              hostid:指定特定主机，可以是ip地址和域名, 也可以指定具体协议，端口。
````
## 功能描述

jps是用于查看有权访问的hotspot虚拟机的进程. 当未指定hostid时，默认查看本机jvm进程，否者查看指定的hostid机器上的jvm进程，此时hostid所指机器必须开启jstatd服务。 jps可以列出jvm进程lvmid，主类类名，main函数参数, jvm参数，jar名称等信息。

![JPS工具的主要选项](/images/jvm/JPS工具的主要选项.png)

例子：

- 没添加option的时候，默认列出VM标示符号和简单的class或jar名称.如下:

![jps无参数](/images/jvm/jps无参数.png)

- -q 仅仅显示VM 标示，不显示jar,class, main参数等信息.

![jps-q命令](/images/jvm/jps-q命令.png)

- -m 输出主函数传入的参数. 下的hello 就是在执行程序时从命令行输入的参数

![jps-m的命令](/images/jvm/jps-m的命令.png)

- -l: 输出应用程序主类完整package名称或jar完整名称.

![jps-m命令](/images/jvm/jps-m命令.png)

- -v: 列出jvm参数, -Xms20m -Xmx50m是启动程序指定的jvm参数

![jps-v命令](/images/jvm/jps-v命令.png)

# jstat 监视JVM内存工具

jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程 [1] 虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据

![jstat工具的知足要选项](/images/jvm/jstat工具的知足要选项.png)

## 语法结构：

````
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
       
参数解释

Options — 选项，我们一般使用 -gcutil 查看gc情况
vmid    — VM的进程号，即当前运行的java进程号
interval– 间隔时间，单位为秒或者毫秒
count   — 打印次数，如果缺省则打印无数次

````

结果项解释：

````
S0  — Heap上的 Survivor space 0 区已使用空间的百分比
S1  — Heap上的 Survivor space 1 区已使用空间的百分比
E   — Heap上的 Eden space 区已使用空间的百分比
O   — Heap上的 Old space 区已使用空间的百分比
P   — Perm space 区已使用空间的百分比
YGC — 从应用程序启动到采样时发生 Young GC 的次数
YGCT– 从应用程序启动到采样时 Young GC 所用的时间(单位秒)
FGC — 从应用程序启动到采样时发生 Full GC 的次数
FGCT– 从应用程序启动到采样时 Full GC 所用的时间(单位秒)
GCT — 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)

````
- 使用实例1：

````
[root@localhost bin]# jstat -gcutil 25444
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT
 11.63   0.00   56.46  66.92  98.49 162    0.248    6      0.331    0.579
````
- 使用实例2  jstat -gcutil

````
(25444是java的进程号，ps -ef | grep java)

[root@localhost bin]# jstat -gcutil 25444 1000 5
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT
 73.54   0.00  99.04  67.52  98.49    166    0.252     6    0.331    0.583
 73.54   0.00  99.04  67.52  98.49    166    0.252     6    0.331    0.583
 73.54   0.00  99.04  67.52  98.49    166    0.252     6    0.331    0.583
 73.54   0.00  99.04  67.52  98.49    166    0.252     6    0.331    0.583
 73.54   0.00  99.04  67.52  98.49    166    0.252     6    0.331    0.583

我们可以看到，5次young gc之后，垃圾内存被从Eden space区(E)放入了Old space区(O)，并引起了百分比的变化，
导致Survivor space使用的百分比从73.54%(S0)降到0%(S1)。有效释放了内存空间。绿框中，我们可以看到，一次full 
gc之后，Old space区(O)的内存被回收，从99.05%降到67.52%。图中同时打印了young gc和full gc的总次数、总耗时。
而，每次young gc消耗的时间，可以用相间隔的两行YGCT相减得到。每次full gc消耗的时间，可以用相隔的两行FGCT相减得到。
例如红框中表示的第一行、第二行之间发生了1次young gc，消耗的时间为0.252-0.252＝0.0秒。常驻内存区(P)的使用率，
始终停留在98.49%左右，说明常驻内存没有突变，比较正常。如果young gc和full gc能够正常发生，而且都能有效回收内存，
常驻内存区变化不明显，则说明java内存释放情况正常，垃圾回收及时，java内存泄露的几率就会大大降低。但也不能说明一定没有内存泄露。

GCT 是YGCT 和FGCT的时间总和。

````

- 使用实例3 jstat -class pid

````
jstat -class pid:显示加载class的数量，及所占空间等信息。

[root@localhost bin]# ps -ef | grep java
root     25917     1  2 23:23 pts/2    00:00:05 /usr/local/jdk1.5/bin/java -Djava.endorsed.dirs=/usr/local/jakarta-tomcat-5.0.30/common/endorsed -classpath /usr/local/jdk1.5/lib/tools.jar:/usr/local/jakarta-tomcat-5.0.30/bin/bootstrap.jar:/usr/local/jakarta-tomcat-5.0.30/bin/commons-logging-api.jar -Dcatalina.base=/usr/local/jakarta-tomcat-5.0.30 -Dcatalina.home=/usr/local/jakarta-tomcat-5.0.30 -Djava.io.tmpdir=/usr/local/jakarta-tomcat-5.0.30/temp org.apache.catalina.startup.Bootstrap start

````

- 使用实例4   jstat -compiler pid:显示VM实时编译的数量等信息。

````
Compiled Failed Invalid   Time   FailedType FailedMethod

     768      0       0   0.70            0

````

- 使用实例5  jstat –gccapacity :可以显示，VM内存中三代（young,old,perm）对象的使用和占用大小，

如：PGCMN显示的是最小perm的内存使用量，PGCMX显示的是perm的内存最大使用量，PGC是当前新生成的perm内存占用量，PC是但前perm内存占用量。其他的可以根据这个类推， OC是old内纯的占用量。

[root@localhost bin]# jstat -gccapacity 25917

````
NGCMN       640.0
NGCMX       4992.0
NGC         832.0
S0C         64.0
S1C         64.0
EC          704.0
OGCMN       1408.0
OGCMX       60544.0
OGC         9504.0
OC          9504.0                  OC是old内纯的占用量
PGCMN       8192.0                  PGCMN显示的是最小perm的内存使用量
PGCMX       65536.0                 PGCMX显示的是perm的内存最大使用量
PGC         12800.0                 PGC是当前新生成的perm内存占用量
PC          12800.0                 PC是但前perm内存占用量
YGC         164
FGC         6

 ````
- 使用实例6 jstat -gcnew pid: new对象的信息

````
[root@localhost bin]# jstat -gcnew 25917
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
 64.0   64.0   47.4   0.0   2  15   32.0    704.0    145.7    168    0.254
````

- 使用实例7 jstat -gcnewcapacity pid: new对象的信息及其占用量

````
[root@localhost bin]# jstat -gcnewcapacity 25917
 NGCMN  NGCMX   NGC   S0CMX  S0C   S1CMX  S1C   ECMX    EC      YGC   FGC
640.0  4992.0  832.0 64.0   448.0 448.0  64.0   4096.0  704.0  168     6
````
 

- 使用实例8 jstat -gcold pid: old对象的信息。

````
[root@localhost bin]# jstat -gcold 25917
   PC       PU        OC          OU       YGC    FGC    FGCT     GCT
 12800.0  12617.6     9504.0      6561.3   169     6    0.335    0.591

 ````

- 使用实例9 jstat -gcoldcapacity pid:old对象的信息及其占用量。

````
[root@localhost bin]# jstat -gcoldcapacity 25917
OGCMN      OGCMX        OGC         OC       YGC   FGC    FGCT     GCT
1408.0     60544.0      9504.0      9504.0   169     6    0.335    0.591
````
 

- 使用实例10 jstat -gcpermcapacity pid: perm对象的信息及其占用量。

````
[root@localhost bin]# jstat -gcpermcapacity 25917
PGCMN      PGCMX       PGC         PC      YGC   FGC    FGCT     GCT
8192.0    65536.0    12800.0    12800.0   169     6    0.335    0.591
````
 

- 使用实例11 jstat -printcompilation pid:当前VM执行的信息。

````
[root@localhost bin]# jstat -printcompilation -h3  25917 1000 5

每1000毫秒打印一次，一共打印5次，还可以加上-h3每三行显示一下标题。

Compiled  Size  Type Method
     788     73    1 java/io/File <init>
     788     73    1 java/io/File <init>
     788     73    1 java/io/File <init>
Compiled  Size  Type Method
     788     73    1 java/io/File <init>
     788     73    1 java/io/File <init>
````


# jinfo。查看和修改JVM运行参数

jinfo（Configuration Info for Java）的作用是实时地查看和调整虚拟机各项参数。使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参
数的系统默认值，除了去找资料外，就只能使用jinfo的-flag选项进行查询了（如果只限于JDK  1.6或以上版本的话，使用java-XX：+PrintFlagsFinal查看参数默认值也是一个很好的选
择），jinfo还可以使用-sysprops选项把虚拟机进程的System.getProperties（）的内容打印出来。JDK  1.6之后，jinfo在Windows和Linux平台都有提供，并且加入了运行期修改参数的能力，可以使用-flag[+|-]name或者-flag  name=value修改一部分运行期可写的虚拟机参数值。
JDK 1.6中，jinfo对于Windows平台功能仍然有较大限制，只提供了最基本的-flag选项

执行样例：查询CMSInitiatingOccupancyFraction参数值。

```` 
jinfo -flag CMSInitiatingOccupancyFration 1444
-XX: CMSInitiatingOccupancyFration=95
````
# jmap：Java内存映像工具

[引用](http://www.cnblogs.com/myna/p/7573843.html)

![jmap工具的主要选项](/images/jvm/jmap工具的主要选项.png)

jmap（Memory Map for Java）命令用于生成堆转储快照（一般称为heapdump或dump文件）。如果不使用jmap命令，要想获取Java堆转储快照，还有一些比较“暴力”的手段：譬如
在第2章中用过的-XX：+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在OOM异常出现之后自动生成dump文件


## 命令格式：
````
jmap [ option ] pid
jmap [ option ] executable core
jmap [ option ] [server-id@]remote-hostname-or-IP
````

## 参数:
````
option：选项参数，不可同时使用多个选项参数
pid：java进程id，命令ps -ef | grep java获取
executable：产生核心dump的java可执行文件
core：需要打印配置信息的核心文件
remote-hostname-or-ip：远程调试的主机名或ip
server-id：可选的唯一id，如果相同的远程主机上运行了多台调试服务器，用此选项参数标识服务器
````

## options参数
````
heap : 显示Java堆详细信息
histo : 显示堆中对象的统计信息
permstat :Java堆内存的永久保存区域的类加载器的统计信息
finalizerinfo : 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象
dump : 生成堆转储快照
F : 当-dump没有响应时，强制生成dump快照
````

## 实例

- dump

dump堆到文件,format指定输出格式，live指明是活着的对象,file指定文件名

````
[root@localhost jdk1.7.0_79]# jmap -dump:live,format=b,file=dump.hprof 24971
Dumping heap to /usr/local/java/jdk1.7.0_79/dump.hprof ...
Heap dump file created

````
- heap

打印heap的概要信息，GC使用的算法，heap的配置及使用情况，可以用此来判断内存目前的使用情况以及垃圾回收情况

````
[root@localhost jdk1.7.0_79]# jmap -heap 24971
Attaching to process ID 24971, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.79-b02
 
using thread-local object allocation.
Parallel GC with 4 thread(s)
 
Heap Configuration:
   MinHeapFreeRatio = 0
   MaxHeapFreeRatio = 100
   MaxHeapSize      = 4146069504 (3954.0MB)
   NewSize          = 1310720 (1.25MB)
   MaxNewSize       = 17592186044415 MB
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2
   SurvivorRatio    = 8
   PermSize         = 21757952 (20.75MB)
   MaxPermSize      = 85983232 (82.0MB)
   G1HeapRegionSize = 0 (0.0MB)
 
Heap Usage:
PS Young Generation
Eden Space:
   capacity = 517996544 (494.0MB)
   used     = 151567520 (144.54605102539062MB)
   free     = 366429024 (349.4539489746094MB)
   29.26033421566612% used
From Space:
   capacity = 41943040 (40.0MB)
   used     = 0 (0.0MB)
   free     = 41943040 (40.0MB)
   0.0% used
To Space:
   capacity = 40370176 (38.5MB)
   used     = 0 (0.0MB)
   free     = 40370176 (38.5MB)
   0.0% used
PS Old Generation
   capacity = 115343360 (110.0MB)
   used     = 32927184 (31.401809692382812MB)
   free     = 82416176 (78.59819030761719MB)
   28.54709972034801% used
PS Perm Generation
   capacity = 85983232 (82.0MB)
   used     = 54701200 (52.16712951660156MB)
   free     = 31282032 (29.832870483398438MB)
   63.6184506300019% used
 
20822 interned Strings occupying 2441752 bytes.
````
- finalizerinfo

打印等待回收的对象信息

````
[root@localhost jdk1.7.0_79]# jmap -finalizerinfo 24971
Attaching to process ID 24971, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.79-b02
Number of objects pending for finalization: 0
````
Number of objects pending for finalization: 0 说明当前F-QUEUE队列中并没有等待Fializer线程执行finalizer方法的对象

- histo

打印堆的对象统计，包括对象数、内存大小等等。jmap -histo:live 这个命令执行，JVM会先触发gc，然后再统计信息

````
[root@localhost jdk1.7.0_79]# jmap -histo:live 24971 | more
 
 num     #instances         #bytes  class name
----------------------------------------------
   1:        100134       14622728  <constMethodKlass>
   2:        100134       12830128  <methodKlass>
   3:         88438       12708392  [C
   4:          8271       10163584  <constantPoolKlass>
   5:         27806        9115784  [B
   6:          8271        6225312  <instanceKlassKlass>
   7:          6830        5632192  <constantPoolCacheKlass>
   8:         86717        2081208  java.lang.String
   9:          2264        1311720  <methodDataKlass>
  10:         10880         870400  java.lang.reflect.Method
  11:          8987         869888  java.lang.Class
  12:         13330         747264  [[I
  13:         11808         733872  [S
  14:         20110         643520  java.util.concurrent.ConcurrentHashMap$HashEntry
  15:         18574         594368  java.util.HashMap$Entry
  16:          3668         504592  [Ljava.util.HashMap$Entry;
  17:         30698         491168  java.lang.Integer
  18:          2247         486864  [I
  19:          7486         479104  java.net.URL
  20:          8032         453616  [Ljava.lang.Object;
  21:         10259         410360  java.util.LinkedHashMap$Entry
  22:           699         380256  <objArrayKlassKlass>
  23:          5782         277536  org.apache.catalina.loader.ResourceEntry
  24:          8327         266464  java.lang.ref.WeakReference
  25:          2374         207928  [Ljava.util.concurrent.ConcurrentHashMap$HashEntry;
  26:          3440         192640  java.util.LinkedHashMap
  27:          4779         191160  java.lang.ref.SoftReference
  28:          3576         171648  java.util.HashMap
  29:         10080         161280  java.lang.Object

````
jmap -histo:live 24971 | grep com.yuhuo 查询类名包含com.yuhuo的信息

jmap -histo:live 24971 | grep com.yuhuo > histo.txt 保存信息到histo.txt文件

![jmap输出中class-name非自定义类的说明](/images/jvm/jmap输出中class-name非自定义类的说明.png)

- permstat

打印Java堆内存的永久区的类加载器的智能统计信息。对于每个类加载器而言，它的名称、活跃度、地址、父类加载器、它所加载的类的数量和大小都会被打印。此外，包含的字符串数量和大小也会被打印。

````
[root@localhost jdk1.7.0_79]# jmap -permstat 24971
Attaching to process ID 24971, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.79-b02
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness....................................................liveness analysis may be inaccurate ...
class_loader    classes bytes   parent_loader   alive?  type
 
<bootstrap>   3034    18149440      null      live    <internal>
0x000000070a88fbb8  1   3048      null      dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x000000070a914860  1   3064    0x0000000709035198  dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x000000070a9fc320  1   3056    0x0000000709035198  dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x000000070adcb4c8  1   3064    0x0000000709035198  dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x000000070a913760  1   1888    0x0000000709035198  dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x0000000709f3fd40  1   3032      null      dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x000000070923ba78  1   3088    0x0000000709035260  dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x000000070a88fff8  1   3048      null      dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58
0x000000070adcbc58  1   1888    0x0000000709035198  dead    sun/reflect/DelegatingClassLoader@0x0000000703c50b58

````

# jstack：Java堆栈跟踪工具

jstack（Stack  Trace  for  Java）命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈
的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。线程出现停顿
的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者等待着什么资源.

[使用jstack精确找到异常代码](https://blog.csdn.net/mr__fang/article/details/68496248)
[jstack 工具使用](https://www.cnblogs.com/kongzhongqijing/articles/3630264.html)
[性能调优](https://blog.csdn.net/yaowj2/article/category/855894)

# JConsole：Java监视与管理控制台 虚拟机监控工具

## 启动

通过JDK/bin目录下的“jconsole.exe”启动JConsole后，将自动搜索出本机运行的所有虚拟机进程，不需要用户自己再使用jps来查询了






# VisualVM 多合一故障处理工具 









