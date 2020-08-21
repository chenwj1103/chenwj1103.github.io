---
title: JVM调优总结
copyright: true
date: 2018-10-18 23:35:04
tags: JVM调优
categories: JVM

---

# 堆大小设置

对于32位的系统，一般限制在1.5-2G；64位操作系统对内存无限制。

## 典型设置
````
java -Xmx3550m -Xms3550m -Xmn2g -Xss128k

java -Xmx3550m -Xms3550m -Xss128k -XX:NewRatio=4 -XX:SurvivorRatio=4 -XX:MaxPermSize=16m -XX:MaxTenuringThreshold=0

-Xmx3550M 最大堆内存
-Xms3550M 初始内存
-Xmn2G    年轻代内存 这个JVM的大小=年轻代大小+年老代的大小+持久代的大小，持久代一般固定为64M，年轻代越大就会减小年老代的大小，此值对系统的性能影响较大，sun官方推荐配置为整个对的3/8
-Xss128K  线程栈的大小 设置每个线程栈的大小。jdk5以后每个线程堆栈的大小为1M，以前每个线程栈的大小为256k
-XX:NewRatio=4 设置年轻代（包括Eden和Survivor）与老年代的比值（除去持久代）。设置为4 则年轻代与老年代所占比值为1:4，年轻代占整个堆栈的1/5
-XX:SurvivorRatio=4 设置年轻代中的Eden与Survivor区的大小比值。设置为4，则两个Survivor与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6
-XX:MaxPermSize=16M 设置持久代的大小为16M
-XX:MaxtenuringThreshold=0 设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区直接进入年老代。对于年老代比较多的应用，则可以提高效率。如果设置一个较大值，则年轻对象则会在Survivor区进行多次复制，则可以增加对象在年轻代的存活时间。
````
# 回收器选择

JVM给了三种选择：串行收集器/并行收集器/并发收集器。但是串行收集器只适用于小数据量的情况，所以这里的选择主要针对并行收集器和并发收集器。默认情况下，JDK5.0以前都是使用串行收集器，如果想使用其他收集器需要在启动时加入相应参数。JDK5.0以后，JVM会根据当前系统配置进行判断。

## 吞吐量优先的并行收集器

并行收集器主要以到达一定的吞吐量为目标，适用于科学技术和后台处理等

````
java -Xmx3800m -Xms3800m -Xmn2g -Xss128k -XX:+UseParallelGC -XX:ParallelGCThreads=20
-XX:+UseParallelGC:选择垃圾收集器为并行收集器。此配置仅对年轻代有效。即上述配置下，年轻代使用并行收集器，老年代使用串行收集器
-XX：ParallelGCThreads=20：配置并行收集器的线程数，即同时多少个线程一起并行垃圾回收。此值最好与处理器数目相等。

java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseParallelGC -XX:ParallelGCThreads=20 -XX:+UseParallelOldGC
-XX:+UseParallelOldGC 配置年老代垃圾收集方式为并行收集。JDK6支持对年老代并行收集

java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseParallelGC  -XX:MaxGCPauseMillis=100
-XX:+MaxGCPauseMillis=100:设置年轻代垃圾回收的最长时间，如果无法满足此时间，JVM会自动调整年轻代的大小，以蛮子此值。

java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseParallelGC  -XX:MaxGCPauseMillis=100 -XX:+UseAdaptiveSizePolicy
-XX:+UseAdaptiveSizePolicy:设置此值后，并行收集器会自动选择年轻代区大小和相应的Survivor区比例，以达到目标系统规定的最低相应时间和收集频率，此值建议使用并行收集器时，一直打开。

````

## 响应时间有限的并发收集器

并发收集器主要是保证系统的响应时间，减少垃圾收集时的停顿时间。适用于应用服务器、电信领域等。

````
java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:ParallelGCThreads=20 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC
-XX:+UseConcMarkSweepGC 设置老年代为并发收集，配置这个以后 -XX:NewRatio=4的配置失效了，原因不明。所以此时年轻代大小最好使用-Xmn设置
-XX:+UseParNewGC 设置年轻代为并行收集。可与CMS收集同时使用。JDK5以上，JVM会根据系统设置自行配置，无需设置此值。

java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseConcMarkSweepGC -XX:CMSFullGCsBeforeCompaction=5 -XX:+UseCMSCompactAtFullCollection
-XX：CMSFullGCsBeforeCompaction:由于并发收集器不对内存空间压缩整理，所以一段时间以后会产生碎片，使得运行效率低下，此值设置运行多少次GC有多少次以后对内存空间进行压缩整理
-XX：+UseCMSCompactFullCollection:打开年老代的压缩，可能会影响性能，但是可以消除碎片

````
# 辅助信息

JVM提供了许多命令行参数，打印信息，供调试使用

````
-XX:+PrintGC
输出形式：[GC 118250K->113543K(130112K), 0.0094143 secs]
        [Full GC 121376K->10414K(130112K), 0.0650971 secs]

-XX：+PrintGCDetails
输出形式：[GC [DefNew: 8614K->781K(9088K), 0.0123035 secs] 118250K->113543K(130112K), 0.0124633 secs]
        [GC [DefNew: 8614K->8614K(9088K), 0.0000665 secs][Tenured: 112761K->10414K(121024K), 0.0433488 secs] 121376K->10414K(130112K), 0.0436268 secs]

-XX:+PrintGCTimeStamps -XX:+PrintGC：PrintGCTimeStamps可与上面两个混合使用
输出形式：11.851: [GC 98328K->93620K(130112K), 0.0082960 secs]

-XX:+PrintGCApplicationConcurrentTime:打印垃圾回收前，程序未中断的执行时间
输出形式：Application time: 0.5291524 seconds

-XX:+PrintGCApplicationStoppedTime：打印垃圾回收期间程序暂停的时间。可与上面混合使用
输出形式：Total time for which application threads were stopped: 0.0468229 seconds

-XX:PrintHeapAtGC:打印GC前后的详细堆栈信息
输出形式：
34.702: [GC {Heap before gc invocations=7:
 def new generation   total 55296K, used 52568K [0x1ebd0000, 0x227d0000, 0x227d0000)
eden space 49152K,  99% used [0x1ebd0000, 0x21bce430, 0x21bd0000)
from space 6144K,  55% used [0x221d0000, 0x22527e10, 0x227d0000)
  to   space 6144K,   0% used [0x21bd0000, 0x21bd0000, 0x221d0000)
 tenured generation   total 69632K, used 2696K [0x227d0000, 0x26bd0000, 0x26bd0000)
the space 69632K,   3% used [0x227d0000, 0x22a720f8, 0x22a72200, 0x26bd0000)
 compacting perm gen  total 8192K, used 2898K [0x26bd0000, 0x273d0000, 0x2abd0000)
   the space 8192K,  35% used [0x26bd0000, 0x26ea4ba8, 0x26ea4c00, 0x273d0000)
    ro space 8192K,  66% used [0x2abd0000, 0x2b12bcc0, 0x2b12be00, 0x2b3d0000)
    rw space 12288K,  46% used [0x2b3d0000, 0x2b972060, 0x2b972200, 0x2bfd0000)
34.735: [DefNew: 52568K->3433K(55296K), 0.0072126 secs] 55264K->6615K(124928K)Heap after gc invocations=8:
 def new generation   total 55296K, used 3433K [0x1ebd0000, 0x227d0000, 0x227d0000)
eden space 49152K,   0% used [0x1ebd0000, 0x1ebd0000, 0x21bd0000)
  from space 6144K,  55% used [0x21bd0000, 0x21f2a5e8, 0x221d0000)
  to   space 6144K,   0% used [0x221d0000, 0x221d0000, 0x227d0000)
 tenured generation   total 69632K, used 3182K [0x227d0000, 0x26bd0000, 0x26bd0000)
the space 69632K,   4% used [0x227d0000, 0x22aeb958, 0x22aeba00, 0x26bd0000)
 compacting perm gen  total 8192K, used 2898K [0x26bd0000, 0x273d0000, 0x2abd0000)
   the space 8192K,  35% used [0x26bd0000, 0x26ea4ba8, 0x26ea4c00, 0x273d0000)
    ro space 8192K,  66% used [0x2abd0000, 0x2b12bcc0, 0x2b12be00, 0x2b3d0000)
    rw space 12288K,  46% used [0x2b3d0000, 0x2b972060, 0x2b972200, 0x2bfd0000)
}
, 0.0757599 secs]

-Xloggc:filename:与上面几个配合使用，把相关日志信息记录到文件以便分析。

````

# 常见配置汇总

## 堆设置

````
-Xms:初始堆大小
-Xmx:最大堆大小
-XX:NewSize=n:设置年轻代大小
-XX:NewRatio=n:设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
-XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
-XX:MaxPermSize=n:设置持久代大小

````
## 收集器设置
````
-XX:+UseSerialGC:设置串行收集器
-XX:+UseParallelGC:设置并行收集器
-XX:+UseParalledlOldGC:设置并行年老代收集器
-XX:+UseConcMarkSweepGC:设置并发收集器
````

## 垃圾回收统计信息
````
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:filename
````
## 并行收集器设置
````
-XX:ParallelGCThreads=n:设置并行收集器收集时使用的CPU数。并行收集线程数。
-XX:MaxGCPauseMillis=n:设置并行收集最大暂停时间
-XX:GCTimeRatio=n:设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n)
````

## 并发收集器设置

````
-XX:+CMSIncrementalMode:设置为增量模式。适用于单CPU情况。
-XX:ParallelGCThreads=n:设置并发收集器年轻代收集方式为并行收集时，使用的CPU数。并行收集线程数。
````

# 调优总结

## 年轻代大小选择
- 响应时间优先的应用：尽可能设大，直到接近系统的最低响应时间限制（根据实际情况选择）。在此种情况下，年轻代收集发生的频率也是最小的。同时，减少到达年老代的对象。
- 吞吐量优先的应用：尽可能的设置大，可能到达Gbit的程度。因为对响应时间没有要求，垃圾收集可以并行进行，一般适合8CPU以上的应用。

## 年老代大小选择

- 响应时间优先的应用：年老代使用并发收集器，所以其大小需要小心设置，一般要考虑并发会话率和会话持续时间等一些参数。如果堆设置小了，可以会造成内存碎片、高回收频率以及应用暂停而使用传统的标记清除方式；如果堆大了，则需要较长的收集时间。最优化的方案，一般需要参考以下数据获得：

并发垃圾收集信息

持久代并发收集次数

传统GC信息

花在年轻代和年老代回收上的时间比例

减少年轻代和年老代花费的时间，一般会提高应用的效率

- 吞吐量优先的应用：一般吞吐量优先的应用都有一个很大的年轻代和一个较小的年老代。原因是，这样可以尽可能回收掉大部分短期对象，减少中期的对象，而年老代尽存放长期存活对象。

## 较小堆引起的碎片问题
- 因为年老代的并发收集器使用标记、清除算法，所以不会对堆进行压缩。当收集器回收时，他会把相邻的空间进行合并，这样可以分配给较大的对象。但是，当堆空间较小时，运行一段时间以后，就会出现“碎片”，如果并发收集器找不到足够的空间，那么并发收集器将会停止，然后使用传统的标记、清除方式进行回收。如果出现“碎片”，可能需要进行如下配置：
````
-XX:+UseCMSCompactAtFullCollection：使用并发收集器时，开启对年老代的压缩。
-XX:CMSFullGCsBeforeCompaction=0：上面配置开启的情况下，这里设置多少次Full GC后，对年老代进行压缩
````


[深入理解java虚拟机](https://www.cnblogs.com/wangzhongqiu/p/8908266.html)

# 自带的内存管理机制

jconsole JDK自带的内存监测工具，路径jdk bin目录下jconsole.exe，双击可运行。连接方式有两种，第一种是本地方式如调试时运行的进程可以直接连，第二种是远程方式，可以连接以服务形式启动的进程。远程连接方式是：在目标进程的jvm启动参数中添加-Dcom.sun.management.jmxremote.port=1090 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false 1090是监听的端口号具体使用时要进行修改，然后使用IP加端口号连接即可。通过该工具可以监测到当时内存的大小，CPU的使用量以及类的加载，
还提供了手动gc的功能。优点是效率高，速度快，在不影响进行运行的情况下监测产品的运行。缺点是无法看到类或者对象之类的具体信息

# 堆内存的分配策略

- 策略1

当在程序中生成对象时，正常对象会在年轻代中分配空间，如果是过大的对象也可能会直接在年老代生成（据观测在运行某程序时候每次会生成一个十兆的空间用收发消息，这部分内存就会直接在年老代分配）。年轻代在空间被分配完的时候就会发起内存回收，大部分内存会被回收，一部分幸存的内存会被拷贝至Survivor的from区，经过多次回收以后如果from区内存也分配完毕，就会也发生内存回收然后将剩余的对象拷贝至to区。等到to区也满的时候，就会再次发生内存回收然后把幸存的对象拷贝至年老区。

JVM初始分配的内存由-Xms指定，默认是物理内存的1/64；JVM最大分配的内存由 -Xmx指定，默认是物理内存的1/4。默认空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制；空余堆内存大于70%时，JVM会减少堆 直到-Xms的最小限制。因此服务器一般设置-Xms、-Xmx相等以避免在每次GC 后调整堆的大小。
 
- 策略2
 
对象优先在Eden分配，当新生区没有足够的内存是，通过分配担保机制提前转移到老年代中去

大对象直接进入老年代。大对象是指需要大量连续内存空间的对象，虚拟机提供了参数 -XX:PretenureSizeThreshold（只对Serial，PerNew两个回收器起效），令大于这个值得对象直接在老年代分配，避免了Eden和两个Survival之间发生大量的内存复制。

长期存活的对象将进入老年代。虚拟机给每个对象定义了对象年龄计数器（Age），如果对象在Eden出生，经过第一次Minor GC后依然存活，并且能被Survival容纳的话，将被移动到Survival，对象年龄设为1。对象在Survival中每熬过一次Major GC，年龄就增加1，达到一定程度（默认是15），就会被晋升到老年代。对象晋升老年代的阈值，可以通过参数-XX:MaxTenuringThreShold 指定

动态对象年龄判断。如果在Survival空间中相同年龄所有对象的大小综合超过了Survival空间的一半，年龄大于等于这个年龄的对象都会被晋升到老年代。无需等待年龄超过MaxTenuringThreShold指定的年龄

空间分配担保。只要老年代的连续空间大于新生代对象总和或者历次晋升的平均大小，就进行Major GC，否则进行Full  GC。


# JVM常用参数设置

## 整体考虑堆大小

-Xms3550m， 初始化堆大小。通常情况和-Xmx大小设置一样，避免虚拟机频繁自动计算后调整堆大小。 

-Xmx3550m，最大堆大小。

## 考虑分代设置堆大小

首先通过jstat等工具查看应用程序正常情况下需要堆大小，再根据实际情况设置。

### 新生代

-xmn2g，新生代大小。Sun官方推荐配置为整个堆的3/8。 
-XX:SurvivorRatio=8。Eden和Survivor的比值。

### 老年代

老年代=整个堆大小-新生代-永久代

### 永久带

-XX:Permsize=512m,设置永久代初始值。 
-XX:MaxPermsize=512m，设置永久代的最大值。 
注：Java8没有永久代说法，它们被称为元空间，-XX:MetaspaceSize=N

## 考虑虚拟机栈

每个线程池的堆栈大小。在jdk5以上的版本，每个线程堆栈大小为1m，jdk5以前的版本是每个线程池大小为256k。一般设置256k。 
-Xss256K.

## 考虑垃圾收集器

### Serial收集器(串行收集器)

历史最悠久的串行收集器。参数-XX:UseSerialGC。不太常用。

### ParNew和ParOld收集器(并发收集器)

Serial的多线程版本收集器。

### Parallel Scavenge(吞吐量优先垃圾收集器)

````
并行收集器，不同于多线程收集器ParNew，关注吞吐量的收集器。 
-XX:MaxGCPauseMillis=10，设置垃圾收集停顿的最大毫秒数。 
-XX:GCTimeRatio=49，垃圾收集器占比，默认是99。 
-XX:+UseAdaptiveSeizPolicy，GC自适应调节策略。 
-XX:+UseParallelGC,虚拟机Server模式默认值，使用Parallel Scavenge + Serial Old进行内存回收。 
-XX:+UseParallelOldGC, 使用Parallel Scavenge + Parallel Old 进行内存回收。
````

### CMS
````
CMS作为老年代收集器，不能与Parallel Scavenge并存。可能会有内存碎片问题。 
-XX:+UserConcMarkSweepGC，新生代默认用ParNew收集。也可以用-XX:+UserParNewGC强制指定新生代用ParNew收集 
-XX:ParallelGCThreads=4，设置垃圾收集线程数。默认是(CPU数量+3)/4。垃圾收集线程数不少于25%，当CPU数量小于4的时候影响大。 
-XX:CMSInitiatingOccupancyFraction=80，老年代垃圾占比达到这个阈值开始CMS收集，1.6默认是92。设置过高容易导致并发收集失败，会出现SerialOld收集的情况。 
-XX:+UseCMSCompactAtFullCollection，在FULL GC的时候， 对年老代的压缩增加这个参数是个好习惯。可能会影响性能,但是可以消除碎片。 
-XX:CMSFullGCsBeforeCompaction=1，多少次后进行内存压缩。 
-XX:+CMSParallelRemarkEnabled, 为了减少第二次暂停的时间，开启并行remark,降低标记停顿
````

### G1(Garbage First)

````
-XX:+UseG1GC，谨慎使用，需要经过线上测试，还没有被设置为默认垃圾收集器。 
之前的垃圾收集器收集的范围是新生代或者老年代，而G1垃圾收集器收集的范围包括新生代和老年代整个堆。G1将Java堆划为多个大小相同的独立区域(Region)，垃圾收集单位是Region。G1垃圾收集适合至少大于4G内存得系统。并且不会产生内存空间碎片。
````

### 其他参数
````
-XX:MaxTenuringThreshold=30，晋升老年代的年龄。 
-XX:PretenureSizeThreshold=?，晋升老年代的对象大小。没设置过。
````

### 考虑日志打印

````
-verbose:gc，打印GC日志 
-XX:+PrintGC，打印GC基本日志 
-XX:+PrintGCDetails，打印GC详细日志 
-XX:+PrintGCTimeStamps，打印相对时间戳 
-XX:+PrintGCApplicationStoppedTime,打印垃圾回收期间程序暂停的时间 
-XX:+PrintGCApplicationConcurrentTime,打印每次垃圾回收前,程序未中断的执行时间 
-XX:+PrintTenuringDistribution：查看每次minor GC后新的存活周期的阈值 
-XX:+PrintTLAB,查看TLAB空间的使用情况 
-Xloggc:filename,把相关日志信息记录到文件以便分析

````

### 考虑OOM(堆溢出)时保留现场日志

````
当抛出OOM时进行heapdump
-XX:+HeapDumpOnOutOfMemoryError,JVM异常自动生成堆转储 
-XX:HeapDumpPath=，堆转储文件名
````











