---
title: 深入理解java虚拟机-垃圾收集器与内存分配策略
copyright: true
date: 2018-10-24 09:10:25
tags: 垃圾收集器与分配策略
categories: jvm

---

java内存运行区域的各个部分，其中程序计数器、虚拟机栈、本地方法栈三个区域岁线程而生而灭；栈中的栈帧随着方法的进入和退出而有条不紊的执行和出栈和入栈操作，每一个帧栈中分配多少内存基本上在类的结构确定下来的时候就已经确定了，所以这些区域就不需要考虑垃圾回收。
而java堆和方法则不一样，一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存可能不一样，我们只有在程序处于运行期间才知道回创建那些对象，这些内存的分配都是动态的，垃圾收集器所关注的事这部分内存。

# 对象存活判断算法

## 引用计数法

给对象添加一个引用计数器，每当有一个地方引用时加一，当引用失效时建议，为0时对象就不可以引用。优点是效率高，缺点是难以解决循环引用的问题。


例子：
````
/**
 * 循环引用的垃圾回收
 *
 * @author :  chen weijie
 * @Date: 2018-10-24 9:20 AM
 */
public class ForEachReferenceDemo {


    public Object instance = null;
    private static final int _1MB = 1024 * 1024;

    //这个成员变量的唯一作用就是站点内存，一边在GC日志中可以看清楚是否被回收过

    private byte[] bigSize = new byte[2 * _1MB];
    public static void testGC() {
        ForEachReferenceDemo objectA = new ForEachReferenceDemo();
        ForEachReferenceDemo objectB = new ForEachReferenceDemo();
        objectA.instance = objectA;
        objectB.instance = objectB;
        objectA = null;
        objectB = null;
        System.gc();


    }
    public static void main(String[] args) {
        testGC();
    }
}

````

由5263K->648K(19456K) 可以得知内存有回收，所以虚拟机回收的算法并不是使用引用计数算法。

## 可达性分析算法

这个算法的基本思路就是通过称为GC Roots 的对象作为起点，从这些节点作为起点向下搜索，走过的路径称为引用链，当一个对象没有 与任何一用力按相连时，证明对象是不可达的。

- java语言中可以作为GC roots的对象包括以下几种：

1.虚拟机栈（帧栈中的本地变量表）中引用的对象；
2.方法区中类静态变量属性引用的对象；
3.方法区中常量引用的对象；
4.本地方法栈中JNI（一般的native方法）引用的对象；


## 再谈引用

一般的引用是指，reference类型的数据中存储的数值代表的是另外一块内存的起始地址；

细分引用：

1.强引用：代码中普遍存在，Object o = new Object() 。只要强引用存在，垃圾收集器永远不会回收掉被引用的对象；
2.软引用：有用但是非必须的，在系统将要发生内存溢出异常之前，将会把这些对象进行第二次回收；
3.弱引用：非必须对象，但是强度比弱引用更低，被弱引用关联的对象只能生存到下一次垃圾收集发生之前；
4.虚引用：最弱的，完全对垃圾收集不产生影响，只不过在垃圾收集器回收时收到一个系统通知；


## 生存还是死亡

即使在可达性分析算法中不可达的对象，也并非是非死不可的。至少要经过两次标记。

当对象在进行可达性分析后发现对象没有与GC Roots相连时，它将会被进行一次标记和一次筛选。筛选的条件是该对象是否有必要执行finalize()方法，当对象没有覆盖finalize()方法或者finalize()方法已经被虚拟机调用过，这2种情况就是没有必要执行。

如果有必要执行finalize方法，那么对象就会被放置到F-Queen队列中。finalize是对象逃脱死亡的最后机会，只要在此期间重新与任何一个对象发生关联即可，这样他会逃脱回收。

## 方法区回收

方法区（HotSpot虚拟机中的永久带）是没有垃圾收集的，因为他的垃圾收集效率远低于此。因为新生代中的垃圾收集一般可以回收70%-90%的空间。

方法区（永久带）的垃圾收集包括废弃常量和无用类。

### 废弃常量回收

例如一个abc字符串已经进入了常量池中，但是系统没有任何一个String对象引用该字符串。

### 无用类回收

满足以下三个条件：

1.该类的所有的实例都已经被回收，就是java堆中不存在该对象的实例；
2.加载该类的classLoader类已经被回收；
3.该类对应的java.lang.Class对象没有任何地方引用，无法再任何地方通过反射获取该类；

满足以上三个条件可以回收，但是类的回收需要对参数进行设置。

大量的使用反射，动态代理，CGLib等byteCode框架、动态生成JSP的以及OSGi这类频繁自定义classLoader的场景都需要虚拟机聚类类卸载的功能，保证永久带不会溢出；

# 垃圾收集算法

## 标记清除算法（mark-Sweep）

首先标记需要回收的对象，标记完统一回收被标记的对象。不足：一是 效率问题，标记和清除效率都不高；二是 标记清除后会产生大量的不连续的内存空间，无法为较大的对象分配空间。

## 复制算法（copying）

为了解决效率问题，使用的复制算法，由于新生代的对象98%都是“朝生夕死”的，将内存区域划分为eden、from survivor、to survivor区 比例为8:1:1,每次回收都是将eden和一个survivor中的存活的对象复制到另一个survivor中。如果此时另一个survivor区域没有足够的空间时，将对象直接放到老年代。

很明显缺点就是：浪费空间以及当有大量对象需要复制时效率很低；一般新生代使用这种算法

## 标记整理算法（mark-compact）

复制算法在对象存活率较高的情况下需要进行较多的复制，效率会变低。根据老年代的特点，有一种标记整理（mark-compact）算法，标记过程与标记清理算法一样，后续步骤不是直接清理回收对象，而是让存货对象向一段移动，然后清理端边以外的内存。

## 分代收集算法

当前虚拟机的收集算法都是用分代收集。 

新生代：新生代中每次垃圾收集时，都会有大批对象死去，只有少量存活，采用复制算法，只需要付出少量的存活对象的复制成本就可以完成收集；

老年代：老年代存活率比较高，没有额外的空间分配担保，必须使用'标记清理'或者'标记整理'算法回收；

# HotSpot的算法实现

在GC的过程中不可以出现分析过程中对象引用关系还在不断变化的情况，该点不满足的话，准确性就无法保证，所有GC进行时都是需要停顿的（Stop the world）

当执行系统停下来后，并不需要一个不漏的检查完所有的执行上下文和全局引用位置，虚拟机是有办法知道哪些地方存放对象引用。在HotSpot的实现中是使用一组称为OopMap的结构来达到这个目的的。

## 安全点

HotSpot没有为每条指令都生产OopMap，只有在特定的位置记录了这些信息，这些位置被称为安全点（safePoint）,即程序执行时并非在所有地方的都能停顿下来GC，只有到达安全点才会暂停。

有2种方式可以让线程停止到最近的安全点上停顿下来：抢占式中断和主动式中断。

- 抢占式中断不需要线程的执行代码主动去配合，在GC发生时，首先所有线程中断，如果发现线程中断的地方不在安全点，让线程回复，让它跑到安全点上。几乎没有虚拟机采用抢占式中断来暂停线程从而响应GC事件
- 主动式中断的思想是当GC需要中断线程的时候，不直接对线程操作，仅仅简单设置一个标记，各个线程执行时主动去轮训这个标记，发现标记位真时就中断挂起。

## 安全区域 

safePoint似乎近乎完美的解决了如何进入GC的问题，但是如果线程处于Sleep或者blocked状态时，这时候线程无法响应JVM的重拳请求，走到安全的地方挂起，这就需要一个安全区域（safe region）来解决问题。

# 垃圾收集器

![HotSpot虚拟机的垃圾收集器](/images/jvm/HotSpot虚拟机的垃圾收集器.png)

上图展示了HotSpot虚拟机的垃圾收集器，如果两个收集器之间存在连线，则说明他们可以搭配使用，所以你所处的区域咋代表他们属于新生代收集器还是老年代收集器。

虚拟机收集器没有最好的收集器，只有在某种场景下最适合的收集器。

## Serial收集器

Serial收集器是最基本、历史最悠久的收集器，单线程收集器，

单线程收集器并不意味着它只会使用一个CPU或者一条收集线程去完成垃圾收集，重要的是GC时，必须暂停其它所有的工作线程，至到收集完成。

该收集器优点是：它是虚拟机运行在Client模式下的默认新生代收集器。简单而高效，因为对于单个CPU的环境说，Serial收集器由于没有线程交互的开销，他是最高效的。

![Serial收集器](/images/jvm/Serial收集器.png)


## ParNew收集器

ParNew收集器就是Serial收集器的多线程版本，使用多线程收集。

![ParNew收集器](/images/jvm/ParNew收集器.png)

它是运行在Server模式下的虚拟机中首选的新生代垃圾收集器，主要是因为除了Serial只有ParNew是CMS收集器一起使用的垃圾收集器。

ParNew收集器在单CPU的环境中没有Serial好。

- 并行 是指多条垃圾收集线程并行工作。
- 并发 是指工作线程与垃圾收集线程同时执行（可以同时执行，可能是交替执行），用户线程在继续运行，而垃圾收集程序在另一个CPU上运行。

## Parallel Scavenge 收集器

- Parallel Scavenge 收集器是一个新生代收集器，并行的多线程收集器，采用复制算法。这些和ParNew一样。它的关注点与其他收集器不同，CMS等收集器的关注点是尽可能的缩短用户线程的停顿时间，而Parallel Scavenge 收集器的目标则是到达一个可控的吞吐量。吞吐量= 运行用户代码时间/（运行用户代码时间+垃圾回收时间）

- 停顿时间越短就越适合需要与用户交互的程序，可以提高用体验。吞吐量则可以高效的利用CPU时间，尽快完成程序的运算任务，适合在后台运算不需要太多交互的任务。对于同一个垃圾回收器来说：吞吐量和响应时间是相斥的属性。

### 参数设置

- Parallel Scavenge 收集器 使用两个参数用于控制吞吐量：
1. -XX:GCTimeRatio(直接设置吞吐量大小，默认99)、
2. -XX:MAXGCPauseMillis(最大垃圾收集停顿时间)

由于与吞吐量关系密切，Parallel Scavenge收集器也经常称为“吞吐量优先”收集器。除上述两个参数之外，Parallel Scavenge收集器还有一个参数-XX：+UseAdaptiveSizePolicy值得关
注。这是一个开关参数，当这个参数打开之后，就不需要手工指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX：SurvivorRatio）、晋升老年代对象年龄（-XX：
PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种调节方式称为GC自适应的调节策略（GC Ergonomics）


## Serial Old收集器

Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用“标记-整理”算法。这个收集器的主要意义也是在于给Client模式下的虚拟机使用。

如果在Server模式下，那么它主要还有两大用途：一种用途是在JDK 1.5以及之前的版本中与Parallel Scavenge收集器搭配使用，另一种用途就是作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用

![SerialOld收集器](/images/jvm/SerialOld收集器.png)

## Parallel Old收集器

Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。

![Parallel Old收集器与Parallel Scavenge收集器](/images/jvm/Parallel Old收集器与Parallel Scavenge收集器.png)

## CMS收集器

### 特点
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。基于“标记—清除”算法实现

### 运行步骤
他的运行分为四个步骤：

- 初始标记（CMS initial mark）：初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快；
- 并发标记（CMS concurrent mark）：并发标记阶段就是进行GC RootsTracing的过程；
- 重新标记（CMS remark）：重新标记阶段则是为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录
- 并发清除（CMS concurrent sweep）：多个垃圾回收线程一起清除。

整个过程中耗时最长的并发标记和并发清除过程收集器线程都可以与用户线程一起工作，所以用时很短。

![Concurrent Mark Sweep收集器运行示意图](/images/jvm/Concurrent Mark Sweep收集器运行示意图.png)

### 缺点

- CMS收集器对CPU资源非常敏感，虽然不会导致用户线程停顿，但是会因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低

CMS默认启动的回收线程数是（CPU数量+3）/4，也就是当CPU在4个以上时，并发回收时垃圾收集线程不少于25%的CPU资源，并且随着CPU数量的增加而下降。但是当CPU不足4个（譬如2个）时，CMS对用户程序的影响就可能变得很大，如果本来CPU负载就比较大，还分出一半的运算能力去执行收集器线程，就
可能导致用户程序的执行速度忽然降低了50%，其实也让人无法接受

为了解决这种问题，使用了一种增量式并发器，就是在并发标记、清理的时候让GC线程、用户线程交替运行，尽量减少GC线程的独占资源的时间，这样整个垃圾收集的过程会更长，但对用户程序的影响就会显得少一些，也就是速度下降没有那么明显

- CMS收集器无法处理浮动垃圾（Floating  Garbage），可能出现“Concurrent  Mode Failure”失败而导致另一次Full GC的产生

由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生，这一部分垃圾出现在标记过程之后，CMS无法在当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就称为“浮动垃圾”

在JDK  1.6中，CMS收集器的启动阈值已经提升至92%。要是CMS运行期间预留的内存无法满足程序需要，就会出现一次“Concurrent Mode Failure”失败，这时虚拟机将启动后备预案：临时启用Serial Old收集器来
重新进行老年代的垃圾收集，这样停顿时间就很长了。所以说参数-XX：CMSInitiatingOccupancyFraction设置得太高很容易导致大量“Concurrent Mode Failure”失败，性能反而降低。

- CMS是一款基于“标记—清除”算法实现的收集器，就可能想到这意味着收集结束时会有大量空间碎片产生，往往会出现老年代还有很大空间剩余，但是无法找到足够大的连续空间来分配当前对象，不得不提前触发一次FullGC

两种解决方法：
1. CMS收集器提供了一个-XX：+UseCMSCompactAtFullCollection开关参数（默认就是开启的），用于在CMS收集器顶不住要进行FullGC时开启内存碎片的合并整理过程，内存整理的过程是无法并发的，空间碎片问题没有了，但停顿时间不得不变长。
2. 另外一个参数-XX：CMSFullGCsBeforeCompaction，这个参数是用于设置执行多少次不压缩的Full  GC后，跟着来一次带压缩的（默认值为0，表示每次进入Full GC时都进行碎片整理）

## G1收集器

G1（Garbage-First）收集器是当今收集器技术发展的最前沿成果之一,G1是一款面向服务端应用的垃圾收集器。它的使命是替代CMS收集器。

### 与CMS相比的特点：

- 并行与并发：G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU（CPU或者CPU核心）来缩短Stop-The-World停顿的时间，部分其他收集器原本需要停顿Java线程执行的GC动作，G1收集器仍然可以通过并发的方式让Java程序继续执行
- 分代收集：与其他收集器一样，分代概念在G1中依然得以保留。虽然G1可以不需要其他收集器配合就能独立管理整个GC堆，但它能够采用不同的方式去处理新创建的对象和已经存活了一段时间、熬过多次GC的旧对象以获取更好的收集效果。
- 空间整理：G1从整体上来看是基于‘标记-整理算法’，局部（两个region）来看，是基于复制算法来是实现的，无论如何，这两种算法都意味着G1运作期间不会产生内存空间碎片，收集后提供规整的可用内存。
- 可预测的停顿：G1除了追求停顿外，还能建立可预测的停顿时间模型，可以让使用者在一个长度M毫秒的时间段内消耗在垃圾收集上的时间不超过N毫秒。

### 详细描述

使用G1收集器时，Java堆的内存布局就与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔离的了，它们都是一部分Region（不需要连续）的集合

G1收集器之所以能建立可预测的停顿时间模型，是因为它可以有计划地避免在整个Java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时
间，优先回收价值最大的Region（这也就是Garbage-First名称的来由）。这种使用Region划分内存空间以及有优先级的区域回收方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率

在G1收集器中，Region之间的对象引用以及其他收集器中的新生代与老年代之间的对象引用，虚拟机都是使用Remembered Set来避免全堆扫描的

### 运行步骤

- 初始标记（Initial Marking） ：与CMS相同
- 并发标记（Concurrent Marking）：并发标记阶段是从GC Root开始对堆中对象进行可达性分析，找出存活的对象，这阶段耗时较长，但可与用户程序并发执行
- 最终标记（Final Marking）：是为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程Remembered Set Logs里面，最终标记阶段需要把Remembered Set Logs的数据合并到Remembered Set
- 筛选回收（Live Data Counting and Evacuation）：首先对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划

![G1收集器的运行示意图](/images/jvm/G1收集器的运行示意图.png)

如果你现在采用的收集器没有出现问题，那就没有任何理由现在去选择G1，如果你的应用追求低停顿，那G1现在已经可以作为一个可尝试的选择，如果你的应用追求吞吐量，那G1并不会为你带来什么特别的好处

# GC日志的理解

- 典型的日志1

![JVM日志1](/images/jvm/JVM日志1.png)

最前面的数字“33.125：”和“100.667：”代表了GC发生的时间，这个数字的含义是从Java虚拟机启动以来经过的秒数

GC日志开头的“[GC”和“[Full  GC”说明了这次垃圾收集的停顿类型，如果有“Full”，说明这次GC是发生了Stop-The-World的。

![ParNew日志也会产生fullGC日志](/images/jvm/ParNew日志也会产生fullGC日志.png)
这段新生代收集器ParNew的日志也会出现“[Full GC”（这一般是因为出现了分配担保失败之类的问题，所以才导致STW）。

如果是调用System.gc（）方法所触发的收集，那么在这里将显示“[Full GC（System）”

接下来的“[DefNew”、“[Tenured”、“[Perm”表示GC发生的区域，这里显示的区域名称与使用的GC收集器是密切相关的。
例如上面样例所使用的Serial收集器中的新生代名为“Default New  Generation”，所以显示的是“[DefNew”。
如果是ParNew收集器，新生代名称就会变为“[ParNew”，意为“Parallel New Generation”。
如果采用Parallel Scavenge收集器，那它配套的新生代称为“PSYoungGen”，老年代和永久代同理，名称也是由收集器决定的。

后面方括号内部的“3324K-＞152K（3712K）”含义是“GC前该内存区域已使用容量-＞GC后该内存区域已使用容量（该内存区域总容量）”。而在方括号之外的“3324K-＞
152K（11904K）”表示“GC前Java堆已使用容量-＞GC后Java堆已使用容量（Java堆总容量）”。再往后，“0.0025925 secs”表示该内存区域GC所占用的时间，单位是秒。
有的收集器会给出更具体的时间数据，如“[Times：user=0.01 sys=0.00，real=0.02 secs]”，这里面的user、sys和real与Linux的time命令所输出的时间含义一致，分别代表用户态消耗的CPU时间、内核
态消耗的CPU事件和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。

- 典型的日志2

程序运行时配置如下参数:
````
-Xms20M -Xmx20M -Xmn10M -verbose:gc -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+PrintGCTimeStamps
最终，程序输出：


0.070: [GC (Allocation Failure) [PSYoungGen: 7127K->616K(9216K)] 11223K->4720K(19456K), 0.0008663 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.072: [GC (Allocation Failure) --[PSYoungGen: 6923K->6923K(9216K)] 11027K->15123K(19456K), 0.0016749 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
0.073: [Full GC (Ergonomics) [PSYoungGen: 6923K->0K(9216K)] [ParOldGen: 8200K->6660K(10240K)] 15123K->6660K(19456K), [Metaspace: 2559K->2559K(1056768K)], 0.0044663 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 9216K, used 4404K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 53% used [0x00000000ff600000,0x00000000ffa4d1a0,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 6660K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 65% used [0x00000000fec00000,0x00000000ff281398,0x00000000ff600000)
 Metaspace       used 2565K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 281K, capacity 386K, committed 512K, reserved 1048576K
````

GC日志分析：

````
1、最前面的数字 "0,070" 代表了GC发生的时间，这个数字的含义是从Java虚拟机启动以来经过的秒数
2、GC日志开头的“[GC 和 [Full GC” 说明了这次垃圾收集的停顿类型，而不是用来区分新生代GC还是年老代GC的。
3、PSYoungGen, ParOldGen，PSPermGen表示GC发生的区域，这里显示的区域名称与使用的GC收集器密切相关，不同收集器对于不同区域所显示的名称可能不同。
4、后面方括号内部的 “ 7127K->616K(9216K) ”含义是“GC前该内存区域已使用容量 -> GC后该内存区域已使用容量（该内存区域总容量）”。方括号之外的 11223K->4720K(19456K) 表示GC前java堆已使用容量 -> GC后java堆已使用容量(Java堆总容量)
5、0.0008663 secs表示该内存区域GC所占用的时间，单位是秒。
6、[Times: user=0.00 sys=0.00, real=0.00 secs] 这里面的user、sys、和real与Linux的time命令所输出的时间含义一致。分别代表用户消耗的CPU时间，内存态消耗的CPU时间，和操作从开始到结束所经过的墙钟时间。
````
 
JVM的GC日志的主要参数包括如下几个：
````
-XX:+PrintGC 输出GC日志
-XX:+PrintGCDetails 输出GC的详细日志
-XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
-XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息
-XX:+PrintGCApplicationStoppedTime // 输出GC造成应用暂停的时间
-Xloggc:../logs/gc.log 日志文件的输出路径
````
# 垃圾收集器参数的总结

![垃圾收集器参数1](/images/jvm/垃圾收集器参数1.png)
![垃圾收集器参数2](/images/jvm/垃圾收集器参数2.png)

# 内存分配与回收策略

对象的内存分配，对象主要分配在新生代的Eden区上，如果启动了本地线程分配缓冲，将按线程优先在TLAB上分配。
少数情况下也可能会直接分配在老年代中，分配的规则并不是百分之百固定的，其细节取决于当前使用的是哪一种垃圾收集器组合，还有虚拟机中与内存相关的参数的设置。

以下的虚拟机使用的是在Parallel Scavenge/Parallel Old收集器下的分配策略。

## 对象优先在Eden分配

大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。

虚拟机提供了-XX：+PrintGCDetails这个收集器日志参数，告诉虚拟机在发生垃圾收集行为时打印内存回收日志，并且在进程退出的时候输出当前的内存各区域分配情况。

例子1

以下代码尝试分配3个2MB大小和1个4MB大小的对象，在运行时通过-Xms20M、-Xmx20M、-Xmn10M这3个参数限制了Java堆大小为20MB，不可扩
展，其中10MB分配给新生代，剩下的10MB分配给老年代。-XX：SurvivorRatio=8决定了新生代中Eden区与一个Survivor区的空间比例是8:1，从输出的结果也可以清晰地看到“eden
space  8192K、from  space  1024K、to  space  1024K”的信息，新生代总可用空间为9216KB（Eden区+1个Survivor区的总容量）

- 新生代GC（Minor GC）：指发生在新生代的垃圾收集动作，因为Java对象大多都具备朝生夕灭的特性，所以Minor GC非常频繁，一般回收速度也比较快。

- 老年代GC（Major GC/Full GC）：指发生在老年代的GC，出现了Major GC，经常会伴随至少一次的Minor GC（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行
Major GC的策略选择过程）。Major GC的速度一般会比Minor GC慢10倍以上。

````
/**
 * VM参数：-verbose：gc -Xms20M-Xmx20M-Xmn10M-XX：+PrintGCDetails
 * -XX：SurvivorRatio=8
 */
public class MinorGCTest {
    private static final int _1MB = 1024 * 1024;
    public static void testAllocation() {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation4 = new byte[2 * _1MB]; //出现一次minor GC
    }
    public static void main(String[] args) {
        testAllocation();
    }
}

[GC [PSYoungGen: 7311K->632K(9216K)] 7311K->6776K(19456K), 0.0093838 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
[Full GC [PSYoungGen: 632K->0K(9216K)] [ParOldGen: 6144K->6656K(10240K)] 6776K->6656K(19456K) [PSPermGen: 2752K->2751K(21504K)], 0.0385372 secs] [Times: user=0.05 sys=0.00, real=0.04 secs] 
Heap
 PSYoungGen      total 9216K, used 2213K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 27% used [0x00000000ff600000,0x00000000ff829788,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 6656K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 65% used [0x00000000fec00000,0x00000000ff280040,0x00000000ff600000)
 PSPermGen       total 21504K, used 2758K [0x00000000f9a00000, 0x00000000faf00000, 0x00000000fec00000)
  object space 21504K, 12% used [0x00000000f9a00000,0x00000000f9cb18c0,0x00000000faf00000)

````

## 大对象直接进入老年代

所谓的大对象是指，需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串以及数组（笔者列出的例子中的byte[]数组就是典型的大对象）。大对象对虚拟机
的内存分配来说就是一个坏消息（替Java虚拟机抱怨一句，比遇到一个大对象更加坏的消息就是遇到一群“朝生夕灭”的“短命大对象”，写程序的时候应当避免），经常出现大对象容易
导致内存还有不少空间时就提前触发垃圾收集以获取足够的连续空间来“安置”它们

虚拟机提供了一个-XX：PretenureSizeThreshold参数，令大于这个设置值的对象直接在老年代分配。这样做的目的是避免在Eden区及两个Survivor区之间发生大量的内存复制.

````
/**
 * VM参数：-verbose：gc-Xms20M-Xmx20M-Xmn10M-XX：+PrintGCDetails-XX：SurvivorRatio=8
 * -XX：PretenureSizeThreshold=3145728
 */
public class BigObjectGCDemo {
    private static final int _1MB = 1024 * 1024;
    public static void testPretenureSizeThreshold() {
        byte[] allocation;
        //直接分配在老年代中
        allocation = new byte[4 * _1MB];
    }
    public static void main(String[] args) {
        testPretenureSizeThreshold();
    }
}

 PSYoungGen      total 9216K, used 5263K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 64% used [0x00000000ff600000,0x00000000ffb23c70,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 0K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 0% used [0x00000000fec00000,0x00000000fec00000,0x00000000ff600000)
 PSPermGen       total 21504K, used 2759K [0x00000000f9a00000, 0x00000000faf00000, 0x00000000fec00000)
  object space 21504K, 12% used [0x00000000f9a00000,0x00000000f9cb1c28,0x00000000faf00000)

````
PretenureSizeThreshold参数只对Serial和ParNew两款收集器有效，Parallel  Scavenge收集器不认识这个参数，Parallel  Scavenge收集器一般并不需要设置

## 长期存活的对象将进入老年代

虚拟机给每个对象定义了一个对象年龄（Age）计数器。如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1。对象在Survivor区中
每“熬过”一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁），就将会被晋升到老年代中。对象晋升老年代的年龄阈值，可以通过参数-XX：MaxTenuringThreshold设置

代码实例:
````
/**
 -verbose:gc
 -Xms20M
 -Xmx20M
 -Xmn10M
 -XX:PrintGCDetails
 -XX:SurvivorRatio=8
 -XX:MaxTenuringThreadshold=1
 -XX:+PrintTenuringDistribution
 */
public class LongExistsObjectDemo {
    private static final int _1MB = 1024 * 1024;
    public static void testTenuringThreshold() {
        byte[] allocation1, allocation2, allocation3;
        allocation1 = new byte[_1MB / 4];
        //什么时候进入老年代取决于XX：MaxTenuringThreshold设置
        allocation2 = new byte[4 * _1MB];
        allocation3 = new byte[4 * _1MB];
        allocation3 = null;
        allocation3 = new byte[4 * _1MB];
    }
    public static void main(String[] args) {
        testTenuringThreshold();
    }
}

````
输入日志：
````
Heap
 PSYoungGen      total 9216K, used 5683K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 69% used [0x00000000ff600000,0x00000000ffb8ccc0,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 8192K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 80% used [0x00000000fec00000,0x00000000ff400020,0x00000000ff600000)
 PSPermGen       total 21504K, used 2759K [0x00000000f9a00000, 0x00000000faf00000, 0x00000000fec00000)

````

## 动态对象年龄判定

为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总
和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

## 空间分配担保

在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么Minor GC可以确保是安全的。如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行
一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者HandlePromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC。

新生代使用复制收集算法，但为了内存利用率，只使用其中一个Survivor空间来作为轮换备份，因此当出现大量对象在MinorGC后仍然存活的情况（最极端的情况就是内存回收后新生代中所有对象都存活），就需要老年代进行分配担保，把Survivor无法容纳的对象直接进入老年代。与生活中的贷款担保类
似，老年代要进行这样的担保，前提是老年代本身还有容纳这些对象的剩余空间，一共有多少对象会活下来在实际完成内存回收之前是无法明确知道的，所以只好取之前每一次回收晋升到老年代对象容量的平均大小值作为经验值，与老年代的剩余空间进行比较，决定是否进行Full GC来让老年代腾出更多空间。

如果某次Minor GC存活后的对象突增，远远高于平均值的话，依然会导致担保失败（Handle Promotion Failure）。如果出现了HandlePromotionFailure失败，那就只好在失败后重新发起一次Full GC。虽然担保失败时绕的圈子是最大的，但大部分情况下都还是会将HandlePromotionFailure开关打开，避免Full GC过于频繁。










