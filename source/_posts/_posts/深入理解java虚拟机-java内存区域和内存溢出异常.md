---
title: 深入理解java虚拟机-java内存区域和内存溢出异常
copyright: true
date: 2018-09-29 00:42:51
tags: jvm
categories: jvm

---


# 概述

对于C和C++程序员，他们负责这每一个对象开始到结束的维护责任，需要为对象分配和释放内存。而java程序员在虚拟机自动内存管理机制的帮助下，不再需要为每一个对象写配对的delete/free代码，不容易出现内存泄露和内存溢出的问题，但是一旦你出现内存泄露和溢出的问题，不了解java虚拟机时不容易解决此类问题的。

## 运行时数据区域

![java虚拟机运行时数据区](/images/jvm/java虚拟机运行时数据区.png)

1.程序计数器

   程序计数器（Program Counter Register） 是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里，字节码解释器工作时就是通过改变这个计数器的值来选取下一条执行字节码指令。

   每条线程都有一个独立的程序计数器。多线程是通过线程轮流切换并分配处理器执行的时间方式来实现的，为了线程切换后可以恢复到正确的执行位置，每条线程都需要一个独立的程序计数器。

   如果执行的是java方法，这个计数器记录的是正在执行的虚拟机字节码指令地址。如果是native方法，计数器为空。此内存区域是唯一一个在java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。

2.Java虚拟机栈

   同样是线程私有，描述Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。一个方法对应一个栈帧。

   局部变量表存放了各种基本类型、对象引用和returnAddress类型（指向了一条字节码指令地址）。其中64位长度long 和 double占两个局部变量空间，其他只占一个。

   规定的异常情况有两种：1.线程请求的栈的深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；2.如果虚拟机可以动态扩展，如果扩展时无法申请到足够的内存，就抛出OutOfMemoryError异常。

3.本地方法栈

   和Java虚拟机栈很类似，不同的是本地方法栈为Native方法服务。

4.Java堆

   是Java虚拟机所管理的内存中最大的一块。由所有线程共享，在虚拟机启动时创建。堆区唯一目的就是存放对象实例。虚拟机规范中描述：所有的对象实例以及数组都在java堆中分配内存。

   堆中可细分为新生代和老年代，再细分可分为Eden空间、From Survivor空间、To Survivor空间。

   堆无法扩展时，抛出OutOfMemoryError异常

5.方法区

   所有线程共享，存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

   当方法区无法满足内存分配需求时，抛出OutOfMemoryError

6.运行时常量池

   它是方法区的一部分，Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项是常量池（Const Pool Table），用于存放编译期生成的各种字面量和符号引用。并非预置入Class文件中常量池的内容才进入方法运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用得比较多的便是String类的intern()方法。

   当方法区无法满足内存分配需求时，抛出OutOfMemoryError

7.直接内存

   并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。

   JDK1.4加入了NIO，引入一种基于通道与缓冲区的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。因为避免了在Java堆和Native堆中来回复制数据，提高了性能。

   当各个内存区域总和大于物理内存限制，抛出OutOfMemoryError异常。

## 对象的创建

创建对象（克隆/反序列化）通常仅仅是一个new关键字而已。虚拟机遇到一条new指令的时候，首先去检查这个指令的参数是否在常量池找那个定位到一个类的符号引用，并检查这个符号引用代表的类是否在内存中已经被加载、解析和初始化。

- 分配内存的方式：

1. 指针碰撞：假设java堆中的内存是绝对规整的，所有用过的内存放到一边，所有没有用过的内存放到另一边，中间放着一个指针作为分界点的指示器，那么内存分配仅仅是把那个指针向空闲内存那的那一边移动与对象大小相等的距离。

2. 空闲列表：如果java堆中的内存并不是规整的，已使用的和未使用的是相互交错的，这样虚拟机必须维护一个记录表，记录哪些内存是可以使用的，在分配内存的时候从列表中找到一块足够打的空间划分给实例，并更新列表上的记录。

而选择哪种方式是由java堆是否规整决定的，而java堆是否规整是由java的垃圾回收器决定的。使用serial、parNew等代用compact过程的收集器时，通常采用的是指针碰撞，而使用CMS这种使用MARK——Sweep算法的收集器时通常采用空闲列表的算法。

- 内存分配的线程安全问题

如果仅仅修改一个指针的指向分配内存，并发情况下的分配会出现问题，解决问题的方案：

1. 对分配内存空间的动作进行同步，实际上是采用CAS配上失败重试的方式保证更新操作的原子性；
2. 另一种是把内存分配的动作按照线程规划在不同的空间中进行，内个线程在java杜仲预先分配一小块内存，称为本地线程分配缓冲(Thread Local Allocation Buffer TLAB),哪个线程需要分配内存就在其对应的TLAB上分配，只用TLAB用完需要分配新的TLAB的时候菜户同步锁定，虚拟机是否使用TLAB可以使用-XX:+UseTLAB参数来决定；

- 分配完之后，虚拟机需要将分配到内存空间都初始化为零值。

- 接下来虚拟机要对对象进行必要的设置，例如这个对象是哪个类的实例，如何才能找到对象的元数据信息，对象的hash码，对象的GC分代年龄等信息。

## 对象的内存布局

在HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头、实例数据、对齐填充。

### 对象头

对象头包括2部分信息：

1. 第一部分存储对象自身的运行时数据，如hash码，GC分代年龄，锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等；
2. 对象头的另一部分是类型指针，即对象指向它的类数据的指针，虚拟机通过这个指针能够确定对象属于哪个类的实例；

### 实例数据

主要存储的是对象真正的有效信息，也就是在程序代码中定义的各种类型的字段内容，无论是从父类继承下来的，还是在子类中定义的都要记录下来。

### 对齐填充

这部分不是必须存在的，也没有特别的含义。它仅仅起占位符的作用，对于HotSpot VM的自动内存管理系统要求对象必须是8字节的整数倍，而对象头的大小正好是8字节的整数倍，因此对象头不需要填充来补全。

## 对象的访问定位

对象是通过栈上的reference数据来操作堆上的具体对象。目前主流的访问方式有句柄和直接指针两种。

### 句柄访问方式

使用句柄访问的话，java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，句柄中保函了对象实例数据与类型数据各自的具体地址信息。

![通过句柄访问对象](/images/jvm/通过句柄访问对象.jpeg)

### 通过指针访问

如果通过指针访问，那么java堆对象的布局就必须考虑如何放置访问类型数据的相关信息。而reference中存储的直接就是对象地址。

![通过指针访问对象](/images/jvm/通过指针访问对象.jpeg)

### 对象访问方式的对比

使用句柄来访问的最大好处就是reference中存储的是稳定的句柄地址，在对象被移动（垃圾回收时移动对象是非常普遍的数据）时改变句柄中的实例数据指针，而reference并不会被改变，

使用直接指针访问的最大好处就是速度快，它节省了一次指针丁文的时间开销。

## 实战OutOfMemoryError异常

虚拟机内存的其他几个运行区域都除了程序计数器都有发生异常OOM的可能。

java 堆用于存储对象实例，只要不断的创建，并且保证GC ROOTs到对象之间有可达路径来避免垃圾回收机制清楚这些对象，那么对象达到最大堆的容量限制后就会产生内存溢出异常。

````

/**
 * -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+HeapDumpOnOutOfMemoryError
 * @author :  chen weijie
 * @Date: 2018-10-23 9:20 AM
 */
public class StackOverFlowErrorDemo {

    static class OOMObject {
    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        while (true) {
            list.add(new OOMObject());
        }
    }
}

````

运行结果如下：

````
/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/bin/java -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:50373,suspend=y,server=n -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+HeapDumpOnOutOfMemoryError -javaagent:/Users/zhuningning/Library/Caches/IntelliJIdea2017.3/captureAgent/debugger-agent.jar=/private/var/folders/52/sbvwq6v93ns69v8s34_drz9h0000gn/T/capture341.props -Dfile.encoding=UTF-8 -classpath "/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/lib/tools.jar:/Users/zhuningning/ideaWorkSpace/javaBase/target/classes:/Users/zhuningning/.m2/repository/org/dom4j/dom4j/2.0.0/dom4j-2.0.0.jar:/Users/zhuningning/.m2/repository/jaxen/jaxen/1.1.6/jaxen-1.1.6.jar:/Users/zhuningning/.m2/repository/junit/junit/4.12/junit-4.12.jar:/Users/zhuningning/.m2/repository/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3.jar:/Users/zhuningning/.m2/repository/org/mongodb/mongodb-driver-async/3.2.2/mongodb-driver-async-3.2.2.jar:/Users/zhuningning/.m2/repository/org/mongodb/mongodb-driver-core/3.2.2/mongodb-driver-core-3.2.2.jar:/Users/zhuningning/.m2/repository/org/mongodb/bson/3.2.2/bson-3.2.2.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-context/4.3.5.RELEASE/spring-context-4.3.5.RELEASE.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-aop/4.3.5.RELEASE/spring-aop-4.3.5.RELEASE.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-beans/4.3.5.RELEASE/spring-beans-4.3.5.RELEASE.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-core/4.3.5.RELEASE/spring-core-4.3.5.RELEASE.jar:/Users/zhuningning/.m2/repository/commons-logging/commons-logging/1.2/commons-logging-1.2.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-expression/4.3.5.RELEASE/spring-expression-4.3.5.RELEASE.jar:/Users/zhuningning/.m2/repository/org/apache/logging/log4j/log4j-api/2.7/log4j-api-2.7.jar:/Users/zhuningning/.m2/repository/org/apache/logging/log4j/log4j-core/2.7/log4j-core-2.7.jar:/Users/zhuningning/.m2/repository/mysql/mysql-connector-java/5.1.40/mysql-connector-java-5.1.40.jar:/Users/zhuningning/.m2/repository/cn/afterturn/easypoi-base/3.0.3/easypoi-base-3.0.3.jar:/Users/zhuningning/.m2/repository/org/apache/poi/poi/3.15/poi-3.15.jar:/Users/zhuningning/.m2/repository/commons-codec/commons-codec/1.10/commons-codec-1.10.jar:/Users/zhuningning/.m2/repository/org/apache/commons/commons-collections4/4.1/commons-collections4-4.1.jar:/Users/zhuningning/.m2/repository/org/apache/poi/poi-ooxml/3.15/poi-ooxml-3.15.jar:/Users/zhuningning/.m2/repository/com/github/virtuald/curvesapi/1.04/curvesapi-1.04.jar:/Users/zhuningning/.m2/repository/org/apache/poi/poi-ooxml-schemas/3.15/poi-ooxml-schemas-3.15.jar:/Users/zhuningning/.m2/repository/org/apache/xmlbeans/xmlbeans/2.6.0/xmlbeans-2.6.0.jar:/Users/zhuningning/.m2/repository/stax/stax-api/1.0.1/stax-api-1.0.1.jar:/Users/zhuningning/.m2/repository/com/google/guava/guava/16.0.1/guava-16.0.1.jar:/Users/zhuningning/.m2/repository/org/apache/commons/commons-lang3/3.2.1/commons-lang3-3.2.1.jar:/Users/zhuningning/.m2/repository/org/slf4j/slf4j-api/1.6.1/slf4j-api-1.6.1.jar:/Users/zhuningning/.m2/repository/javax/validation/validation-api/1.1.0.Final/validation-api-1.1.0.Final.jar:/Users/zhuningning/.m2/repository/cn/afterturn/easypoi-web/3.0.3/easypoi-web-3.0.3.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-web/3.1.1.RELEASE/spring-web-3.1.1.RELEASE.jar:/Users/zhuningning/.m2/repository/aopalliance/aopalliance/1.0/aopalliance-1.0.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-webmvc/3.1.1.RELEASE/spring-webmvc-3.1.1.RELEASE.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-asm/3.1.1.RELEASE/spring-asm-3.1.1.RELEASE.jar:/Users/zhuningning/.m2/repository/org/springframework/spring-context-support/3.1.1.RELEASE/spring-context-support-3.1.1.RELEASE.jar:/Users/zhuningning/.m2/repository/cn/afterturn/easypoi-annotation/3.0.3/easypoi-annotation-3.0.3.jar:/Users/zhuningning/.m2/repository/javax/servlet/javax.servlet-api/3.1.0/javax.servlet-api-3.1.0.jar:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar" com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo
Connected to the target VM, address: '127.0.0.1:50373', transport: 'socket'
[GC (Allocation Failure)  7699K->3961K(19456K), 0.0045637 secs]
[GC (Allocation Failure)  12153K->9254K(19456K), 0.0079381 secs]
[Full GC (Ergonomics)  9254K->9133K(19456K), 0.1111956 secs]
[Full GC (Ergonomics)  17325K->14883K(19456K), 0.1398179 secs]
[Full GC (Ergonomics)  16577K->16432K(19456K), 0.1255392 secs]
[Full GC (Allocation Failure)  16432K->16415K(19456K), 0.1088586 secs]
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid731.hprof ...
Heap dump file created [28119578 bytes in 0.146 secs]
Exception in thread "main" Disconnected from the target VM, address: '127.0.0.1:50373', transport: 'socket'
java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:265)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:239)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:231)
	at java.util.ArrayList.add(ArrayList.java:462)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.main(StackOverFlowErrorDemo.java:21)

Process finished with exit code 1

````

异常堆栈信息 java.lang.OutOfMemoryError 紧接着回提示 java heap space

一般是用内存分析工具检查是否是内存泄漏，如果是内存泄漏则可以查看泄漏对到GC roots的因用力按，于是就找到了泄漏对象是通过怎样的路径与roots相关联的导致垃圾收集器无法自动回收他们；

如果不是内存泄漏（内存中的对象确实都还必须存活的），那就要检查虚拟机的堆参数，与机器物理内存对比看是否可以调大，从代码上检查是否在某些对象生命周期过长，持有有状态时间过长的情况，尝试减少程序运行的内存消耗。

## 虚拟机栈和本地方法栈溢出

使用 -Xss参数设置栈容量

如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常；如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常；


````
package com.chen.api.util.jvm.chapter2;

/**
 * 虚拟机栈和本地方法栈内存溢出测试
 *
 * -verbose:gc -Xms20M -Xmx20M -Xmn10M -Xss300k
 * @author :  chen weijie
 * @Date: 2018-10-23 10:00 AM
 */
public class StackOverFlowErrorDemo {

    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) throws Throwable {
        StackOverFlowErrorDemo oom = new StackOverFlowErrorDemo();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}


````


运行结果如下：

````
stack length:2788
Exception in thread "main" java.lang.StackOverflowError
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
	at com.chen.api.util.jvm.chapter2.StackOverFlowErrorDemo.stackLeak(StackOverFlowErrorDemo.java:15)
````

可以看出抛出栈内存溢出的异常，在单个线程下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常。

如果是建立过多线程导致内存溢出，在不能减少线程数或者更换64位虚拟机的情况下，就只能通过减少最大堆和减少栈容量来换取更多的线程。




### 创建线程导致内存溢出

````

/**
 * 创建线程导致内存溢出异常
 * 需要将线程栈设置的大些  -Xss2M
 * @author :  chen weijie
 * @Date: 2018-10-23 10:22 AM
 */
public class OutOfMemoryErrorForEachDemo {

    int i =0;

    private void dontStop(){
        while (true){
            i++;
            System.out.println(i);
        }
    }

    public void stackLeakByThread(){
        while (true){
            Thread thread =new Thread(new Runnable() {
                @Override
                public void run() {
                    dontStop();
                }
            });
            thread.start();
        }
    }

    public static void main(String[] args) {
        OutOfMemoryErrorForEachDemo oom = new OutOfMemoryErrorForEachDemo();
        oom.stackLeakByThread();
    }
}
````


运行结果：

````

1745594
1745595
1745596
1745597
Oct 23, 2018 10:30:55 AM sun.rmi.transport.tcp.TCPTransport$AcceptLoop executeAcceptLoop
WARNING: RMI TCP Accept-0: accept loop for ServerSocket[addr=0.0.0.0/0.0.0.0,localport=51751] throws
java.lang.OutOfMemoryError: unable to create new native thread
	at java.lang.Thread.start0(Native Method)
	at java.lang.Thread.start(Thread.java:717)
	at java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:957)
	at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1378)
	at sun.rmi.transport.tcp.TCPTransport$AcceptLoop.executeAcceptLoop(TCPTransport.java:415)
	at sun.rmi.transport.tcp.TCPTransport$AcceptLoop.run(TCPTransport.java:372)
	at java.lang.Thread.run(Thread.java:748)

1745598
````

## 方法区和运行时常量池溢出

由于运行时常量池是方法区的一部分，所以放在一起测试进行。

### 运行时常量池内存溢出

String.intern（）方法是一个native方法，它的作用是:如果字符串常量池中已经包含一个等于String对象的字符串，则返回代表池中的字符串的String对象；否则将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。

````
/**
 * -verbose:gc -Xms20M -Xmx20M -Xmn10M -Xss2M -XX:PermSize=10M -XX:MaxPermSize=10M
 * @author :  chen weijie
 * @Date: 2018-10-23 11:09 AM
 */
public class DistantsPoolMemoryStackOverflowDemo {


    public static void main(String[] args) {
        //使用list保持常量池引用，避免full GC回收常量池行为
        List<String> list = new ArrayList<>();
        int i = 0;
        for (; ; ) {
            list.add(String.valueOf(i++).intern());
        }
    }
}

````

java.lang.OutOfMemoryError: GC overhead limit exceeded 这种情况发生的原因是, 程序基本上耗尽了所有的可用内存, GC也清理不了

运行结果：

````
[Full GC (Ergonomics)  18393K->18393K(19456K), 0.0671862 secs]
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
[Full GC (Ergonomics)  18410K->602K(19456K), 0.0118358 secs]
	at java.lang.Integer.toString(Integer.java:401)
	at java.lang.String.valueOf(String.java:3099)
	at com.chen.api.util.jvm.chapter2.DistantsPoolMemoryStackOverflowDemo.main(DistantsPoolMemoryStackOverflowDemo.java:18)
Disconnected from the target VM, address: '127.0.0.1:52849', transport: 'socket'

````

### 方法区内存溢出

方法区用于存储class的相关信息，如类名、访问修饰符、常量池、字段描述等。

当前很多主流的框架如spring，在对类进行增强时都使用到CGLib这类字节码技术，增强的类越多，就需要越大的方法区来保证动态生成的class加入到内存。

方法：

````

````

运行结果：

````
E:\workSoftware\jdk\jdk1.7.0_80\bin\java -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:62719,suspend=y,server=n -verbose:gc -Xms20M -Xmx20M -Xmn10M -Xss2M -XX:PermSize=10M -XX:MaxPermSize=10M -Dfile.encoding=UTF-8 -classpath E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\charsets.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\deploy.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\access-bridge-64.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\dnsns.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\jaccess.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\localedata.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\sunec.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\sunjce_provider.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\sunmscapi.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\ext\zipfs.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\javaws.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\jce.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\jfr.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\jfxrt.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\jsse.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\management-agent.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\plugin.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\resources.jar;E:\workSoftware\jdk\jdk1.7.0_80\jre\lib\rt.jar;F:\ideaWorkSpace\JVMTest\target\classes;E:\workSoftware\maven\maven-repository\org\springframework\spring-context\4.3.5.RELEASE\spring-context-4.3.5.RELEASE.jar;E:\workSoftware\maven\maven-repository\org\springframework\spring-aop\4.3.5.RELEASE\spring-aop-4.3.5.RELEASE.jar;E:\workSoftware\maven\maven-repository\org\springframework\spring-beans\4.3.5.RELEASE\spring-beans-4.3.5.RELEASE.jar;E:\workSoftware\maven\maven-repository\org\springframework\spring-core\4.3.5.RELEASE\spring-core-4.3.5.RELEASE.jar;E:\workSoftware\maven\maven-repository\commons-logging\commons-logging\1.2\commons-logging-1.2.jar;E:\workSoftware\maven\maven-repository\org\springframework\spring-expression\4.3.5.RELEASE\spring-expression-4.3.5.RELEASE.jar;E:\workSoftware\idea\lib\idea_rt.jar com.sunlands.test.CGLibStackOverflowError
Connected to the target VM, address: '127.0.0.1:62719', transport: 'socket'
[GC 8192K->1427K(19456K), 0.0024569 secs]
[GC 9619K->1437K(19456K), 0.0010156 secs]
[GC 9629K->1493K(19456K), 0.0010742 secs]
[GC 9685K->1669K(19456K), 0.0021826 secs]
[GC 9861K->1813K(19456K), 0.0011027 secs]
[GC 10005K->2029K(18432K), 0.0013455 secs]
[GC 9197K->2245K(18944K), 0.0012436 secs]
[GC 9413K->2413K(18944K), 0.0012815 secs]
[GC 9581K->2477K(17920K), 0.0009801 secs]
[GC 9645K->2621K(18432K), 0.0009705 secs]
[GC 8765K->2757K(18432K), 0.0007623 secs]
[GC 8901K->2809K(18432K), 0.0004651 secs]
[GC 8953K->2945K(18432K), 0.0005616 secs]
[GC 9089K->3069K(18432K), 0.0004116 secs]
[GC 9213K->3181K(18432K), 0.0005763 secs]
[GC 9325K->3245K(18432K), 0.0128357 secs]
[GC 9389K->3341K(18432K), 0.0029091 secs]
[GC 9485K->3509K(18432K), 0.0004798 secs]
[GC 9653K->3549K(18432K), 0.0004426 secs]
[GC 9693K->3690K(18944K), 0.0005880 secs]
[GC 10858K->3786K(18944K), 0.0005387 secs]
[GC 10954K->3898K(18944K), 0.0005562 secs]
[GC 11066K->4010K(18944K), 0.0006051 secs]
[GC 11178K->4154K(18944K), 0.0005066 secs]
[GC 11322K->4234K(18944K), 0.0004826 secs]
[GC 11402K->4298K(18944K), 0.0008164 secs]
[GC 6751K->4346K(18944K), 0.0005177 secs]
[Full GC 4346K->3505K(18944K), 0.0425954 secs]
[GC 3505K->3505K(18944K), 0.0003107 secs]
[Full GC 3505K->3505K(18944K), 0.0105272 secs]
[GC 3505K->3505K(18944K), 0.0003374 secs]
[Full GC 3505K->1687K(18944K), 0.0204737 secs]
[GC 1687K->1687K(19456K), 0.0003900 secs]
[Full GC 1687K->1686K(19456K), 0.0088587 secs]
[GC 1851K->1750K(19456K), 0.0003633 secs]
[Full GCException in thread "main"  1750K->1682K(19456K), 0.0109055 secs]
[GC 1682K->1682K(19456K), 0.0002975 secs]
[Full GC 1682K->1682K(19456K), 0.0092689 secs]
[GC 1846K->1778K(19456K), 0.0003080 secs]
[Full GC 1778K->1650K(19456K), 0.0214303 secs]
[GC 1650K->1650K(19456K), 0.0002951 secs]
[Full GC 1650K->1650K(19456K), 0.0097328 secs]
[GC 1650K->1650K(19456K), 0.0002996 secs]
[Full GC 1650K->1650K(19456K), 0.0082941 secs]
[GC 1650K->1650K(19456K), 0.0003107 secs]
[Full GCDisconnected from the target VM, address: '127.0.0.1:62719', transport: 'socket'

Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "main"
 1650K->1650K(19456K), 0.0098416 secs]

````


CGLib字节码增强和动态语言以及JSP文件生产的java类文件都需要注意累的回收情况。


### 本机直接内存溢出

DirectMemory容量可以通过-XX:MaxDirectoryMemorySize指定，如果不指定这与java最大对内存一直，代码直接通过反射火族Unsafe实例进行内存分类。因此虽使用DirectBuffer分配内存也会抛出内存溢出异常，但它抛出异常并没有真正向操作系统申请分配内存，而是通过计算得知内存无法分配，于是手动抛出异常。

