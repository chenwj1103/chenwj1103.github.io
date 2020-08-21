---
title: 多线程编程核心技术-java多线程基本方法
copyright: true
date: 2018-04-08 00:50:28
tags: java多线程基本方法
categories: 多线程

---

# 进程和线程


## 概念

**进程**: 进程是操作系统结构的基础,是一次程序的运行;是一个程序及其数据在处理机上顺序执行所发生的活动;是程序在一个数据集合上运行的过程,是系统进行资源分配和调度的独立单位.

指在系统中能独立运行并作为资源分配的基本单位，它是由一组机器指令、数据和堆栈等组成的，是一个能独立运行的活动实体。

**线程**: 进程间独立运行的子任务.是程序执行流的最小单元

## 区别

1. 调度：
        在传统的操作系统中，CPU调度和分派的基本单位是进程。而在引入线程的操作系统中，则把线程作为CPU调度和分派的基本单位，进程则作为资源拥有的基本单位，从而使传统进程的两个属性分开，线程编程轻装运行，这样可以显著地提高系统的并发性。同一进程中线程的切换不会引起进程切换，从而避免了昂贵的系统调用，但是在由一个进程中的线程切换到另一进程中的线程，依然会引起进程切换。
 
2. 并发性：
      在引入线程的操作系统中，不仅进程之间可以并发执行，而且在一个进程中的多个线程之间也可以并发执行，因而使操作系统具有更好的并发性，从而更有效地提高系统资源和系统的吞吐量。例如，在一个为引入线程的单CPU操作系统中，若仅设置一个文件服务进程，当它由于某种原因被封锁时，便没有其他的文件服务进程来提供服务。在引入线程的操作系统中，可以在一个文件服务进程设置多个服务线程。当第一个线程等待时，文件服务进程中的第二个线程可以继续运行；当第二个线程封锁时，第三个线程可以继续执行，从而显著地提高了文件服务的质量以及系统的吞吐量。

3. 拥有资源：
      不论是引入了线程的操作系统，还是传统的操作系统，进程都是拥有系统资源的一个独立单位，他可以拥有自己的资源。一般地说，线程自己不能拥有资源（也有一点必不可少的资源），但它可以访问其隶属进程的资源，亦即一个进程的代码段、数据段以及系统资源（如已打开的文件、I/O设备等），可供同一个进程的其他所有线程共享。

4. 独立性：
        在同一进程中的不同线程之间的独立性要比不同进程之间的独立性低得多。这是因为
为防止进程之间彼此干扰和破坏，每个进程都拥有一个独立的地址空间和其它资源，除了共享全局变量外，不允许其它进程的访问。但是同一进程中的不同线程往往是为了提高并发性以及进行相互之间的合作而创建的，它们共享进程的内存地址空间和资源，如每个线程都可以访问它们所属进程地址空间中的所有地址，如一个线程的堆栈可以被其它线程读、写，甚至完全清除。

5. 系统开销：
       由于在创建或撤销进程时，系统都要为之分配或回收资源，如内存空间、I/O设备等。因此，操作系统为此所付出的开销将显著地大于在创建或撤消线程时的开销。类似的，在进程切换时，涉及到整个当前进程CPU环境的保存环境的设置以及新被调度运行的CPU环境的设置，而线程切换只需保存和设置少量的寄存器的内容，并不涉及存储器管理方面的操作，可见，进程切换的开销也远大于线程切换的开销。此外，由于同一进程中的多个线程具有相同的地址空间，致使他们之间的同步和通信的实现也变得比较容易。在有的系统中，现成的切换、同步、和通信都无需操作系统内核的干预。

6. 支持多处理机系统：
       在多处理机系统中，对于传统的进程，即单线程进程，不管有多少处理机，该进程只能运行在一个处理机上。但对于多线程进程，就可以将一个进程中的多个线程分配到多个处理机上，使它们并行执行，这无疑将加速进程的完成。因此，现代处理机OS都无一例外地引入了多线程。


# 多线程的实现及基本方法

继承Thread类和实现Runnable接口,thread类也实现了Runnable接口,所以本质上时一样的.但Runnable接口是避免了单继承的缺点.

使用多线程时,代码的调用顺序或者使用顺序和代码的运行结果是无关的.

线程的启动顺序,不代表线程的执行顺序.

## 实例变量与线程安全

自定义线程类中的实例变量针对其它线程可以有共享和不共享之分

1. 不共享的线程是每个线程有自己运行的代码块.
2. 共享的线程的情况就是多个线程访问同一代码块中的变量.

由于多线程共享一段代码的时候(调用run方法),会出现各个线程顺序不定的访问代码.为了实现排队访问同一变量的目的,可以在方法前添加synchronize关键字.

//在执行run方法前,先判断方法是否加synchronize锁,如果上锁,说明其它线程在调用run方法,必须等待其它线程对run方法调用结束后才可以执行run方法

````
public class ShareVariableThread2 extends Thread {

    private int i = 10;

    @Override
    synchronized public void run() {
        i--;
        System.out.println(Thread.currentThread().getName() + "计算,count=" + i);
    }
}



    ShareVariableThread2 thread2 =new ShareVariableThread2();
    Thread t5 =new Thread(thread2,"t5");
    Thread t6 =new Thread(thread2,"t6");
    Thread t7 =new Thread(thread2,"t7");
    Thread t8 =new Thread(thread2,"t8");
    t5.start();
    t6.start();
    t7.start();
    t8.start();
        

````

### 非线程安全

非线程安全是指,多线程对同一个对象的同一个变量进行操作的时候会出现值被更改,值不同步的情况,进而影响程序执行的进程.

currentThread()方法 判断正在被哪个线程调用

isAlive()判断线程是否处于活跃状态

sleep()方法使得线程放弃cpu使用时间，N ms

getId()获取线程的唯一id

## 停止线程的三种方法

stop()是可以停止一个线程的运行，但是它是线程不安全的，它是不推荐使用的方法，在将来的发布版本中可能被去掉，可能产生不可预料的结果。

interrupt()方法，不会终止一个正在运行的线程。需要加入一个判断才可以完成线程的停止。

正常的run方法完成后线程终结。


## 停止不了的线程

调用interrupt()方法仅仅是在当前线程中打了一个停止的标记,并不是真正的停止线程.

````
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }

````
## 判断线程是否处于停止状态

this.interrupted()  测试当前线程是否已经中断,执行后具有将状态标志置为false的功能;

this.isInterrupted() 测试线程是否是中断状态,但不清除状态标志.


## 能停止的线程-异常法

````

/**
 * 通过判断线程是否是停止状态,如果是停止状态,则停止执行后续代码
 *
 * @author chen weijie
 * @date 2018-04-10 12:03 AM
 */
public class StatusStopThread extends Thread {

    @Override
    public void run() {

        for (int i = 0; i < 500000; i++) {

            if (Thread.currentThread().isInterrupted()) {
                break;
            }
            System.out.println("i=========" + i);

        }

    }
}

public class StatusStopThreadTest {

    public static void main(String[] args) {

        StatusStopThread thread = new StatusStopThread();
        thread.start();
        try {
            System.out.println("sleep............");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        thread.interrupt();
        System.out.println("end--------");
    }
}

````
虽然根据线程标记的中断状态,for循环停止了运行,但是'end.....'语句还是输出了,证明线程还是在运行.如何真正中断线程呢? 答案时抛出异常


````

/**
 * 通过抛出异常中断线程
 *
 * @author chen weijie
 * @date 2018-04-10 12:14 AM
 */
public class ExceptionStopThread extends Thread {

    @Override
    public void run() {

        super.run();

        for (int i = 0; i < 5000000; i++) {
            System.out.println("i======" + i);
            if (this.isInterrupted()) {
                System.out.println("中断....");
                throw new RuntimeException("exception interrupt thread");
            }
        }
    }
}

````

## 在沉睡中终止

抛出sleep状态被打断的异常

````
Connected to the target VM, address: '127.0.0.1:33558', transport: 'socket'
beginning............
在沉睡中被中止.......interrupt
end........
Disconnected from the target VM, address: '127.0.0.1:33558', transport: 'socket'
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.chen.api.util.thread.study.chapter1.sleepStopException.SleepInterruptThread.run(SleepInterruptThread.java:16)


````

## 暴力停止线程

暴力终结不会出现在沉睡中interrupt那种抛出异常,非常暴力.

stop被作废,主要是因为,如果强制让线程停止,则有可能使得一些清理性的工作得不到完成.另外一个情况就是对锁定对象进行了"解锁",导致数据得不到同步的处理.


## 释放锁的不良后果

使用stop()释放锁,将会给数据造成不一致的结果,这样程序处理的数据可能遭到破坏.

````
public class SynchronizedObject {

    private String userName ="a";
    private String passWord="aa";
  synchronized  public void printString(String userName,String passWord){

        this.userName =userName;
        try {
            Thread.sleep(100000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.passWord=passWord;
    }
    
    }

public class MyThread extends Thread {
    private SynchronizedObject object;
    public MyThread(SynchronizedObject object) {
        this.object = object;
    }

    @Override
    public void run() {
        object.printString("b", "bb");
    }

}


public static void main(String[] args) {

    try {
        SynchronizedObject object = new SynchronizedObject();
        MyThread myThread = new MyThread(object);
        myThread.start();
        Thread.sleep(500);
        myThread.stop();
        System.out.println(object.getUserName() + "----------" + object.getPassWord());
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}


````

## return 停止线程

return和抛出异常都可以停止线程,但是建议使用抛出异常的方式,因为这样可以在catch块中向上抛,使得线程停止的事件得以传播.

## 暂停线程

suspend()方法是中断线程的,resume()是唤醒线程的.

这两方法的缺点: 

1. 独占,容易造成同步对象的独占,使得其它线程无法访问公共同步对象.
2. 不同步,这两个方法容易出现线程的暂停而导致数据不同步的情况.

## yeild方法

该方法的作用是放弃当前的cpu资源,将它让给其它的任务去占用,但是放弃的时间不确定,所以有可能刚刚放弃,马上又获得cpu的时间了.

## 线程的优先级

cpu会优先执行优先级比较高的线程对象中的任务.所以优先级高的线程会比优先级低的线程获取更多的cpu时间;

cpu尽量尽量将执行资源让给优先级比较高的线程执行.


````
    /**
     * Changes the priority of this thread.
     * <p>
     * First the <code>checkAccess</code> method of this thread is called
     * with no arguments. This may result in throwing a
     * <code>SecurityException</code>.
     * <p>
     * Otherwise, the priority of this thread is set to the smaller of
     * the specified <code>newPriority</code> and the maximum permitted
     * priority of the thread's thread group.
     *
     * @param newPriority priority to set this thread to
     * @exception  IllegalArgumentException  If the priority is not in the
     *               range <code>MIN_PRIORITY</code> to
     *               <code>MAX_PRIORITY</code>.
     * @exception  SecurityException  if the current thread cannot modify
     *               this thread.
     * @see        #getPriority
     * @see        #checkAccess()
     * @see        #getThreadGroup()
     * @see        #MAX_PRIORITY
     * @see        #MIN_PRIORITY
     * @see        ThreadGroup#getMaxPriority()
     */
    public final void setPriority(int newPriority) {
        ThreadGroup g;
        checkAccess();
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
            throw new IllegalArgumentException();
        }
        if((g = getThreadGroup()) != null) {
            if (newPriority > g.getMaxPriority()) {
                newPriority = g.getMaxPriority();
            }
            setPriority0(priority = newPriority);
        }
    }


````
 
 
 线程优先级分为10个 但JDK定义了三个常量: MIN_PRIORITY=1  NORM_PRIORITY=5  MAX_PRIORITY=10 
 
 线程优先级的继承性: 比如A线程启动B线程,则B线程的优先级与A是一样的.
 
 
 ## 优先级具有随机性
 
 优先级较高的线程会获取较多的cpu资源,所以两个具有相同任务的线程优先级不同,优先级高的线程的任务先执行完毕,但不是绝对的,因为线程的优先级具有随机性.
 
 ## 守护线程
 
 java线程中有2种: 一种是守护线程,另一种是用户线程
 
 1. 守护线程时一种特殊的线程,他的特性有陪伴的含义,当进程中不存在非守护线程了,守护进程自动销毁.
 
 2. 任何一个守护线程都是整个JVM中所有非守护线程的保姆.只要当前JVM中存在任何一个非守护线程没有结束,守护线程就在工作,当最后一个非守护线程结束时没守护线程才随着JVM一同结束同坐,
 
 3. Daemon的作用时为其它线程的运行提供遍历服务,守护线程最典型的应用就是GC,他就是一个很称职的守护者.
 
 


