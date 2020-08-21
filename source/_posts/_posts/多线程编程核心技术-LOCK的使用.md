---
title: 多线程编程核心技术-LOCK的使用
copyright: true
date: 2018-05-16 23:42:02
tags: 多线程Lock的使用
categories: 多线程

---

# ReentrantLock的使用

synchronized关键字来实现线程间同步互斥，在jdk1.5之后增加的ReentrantLock锁也能达到同样的效果。而且增加了嗅探锁定，多路分支通知功能，而且比synchronized更加灵活。

## conditions实现等待通知

 synchronized与wait()和notify()、notifyAll() 方法相结合实现等待通知模式，ReentrantLock锁借助于Condition对象也是可以实现多路通知的功能。
 
 实现多路通知的功能，也就是一个Lock对象中可以创建多个Condition实例，线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。
 
 而notify()方法进行通知时，被通知的线程却是由JVM随机选择的。而synchronized就相当于整个LOCK对象中只有一个单一的Condition对象，所有的线程都注册在一个对象身上，线程开始notifyAll()时，需要通知所有waiting线程，没有选择权，会出现相当大的效率问题。
 
 
 
## 公平锁和非公平锁

公平锁表示线程获取锁的顺序是按照线程加锁的顺序来分配的，即先来先得的FIFO先进先出的顺序。而非公平锁就是一种获取锁的抢占式机制，是随机获得锁的。某些线程是一直拿不到锁的，所以是不公平的锁了。

## 方法介绍

- ReentrantLock(boolean) 创建是否公平的锁：ReentrantLock(boolean) 是创建是否公平的锁；
- lock.getHoldCount()  获取当前线程调用lock的次数； 
- lock.getQueenLength() 获取正等待获取此锁定的线程估计数 lock.getQueenLength()
- getWaitQueueLength(Condition condition) 获取等待与此锁定相关的给定条件Condition的线程估计数，比如有5个线程，每个线程都执行了同一个Condition对象的await()方法，调用getWaitQueueLength(Condition condition) 返回的int值是5 
- lock.hasQueuedThread(Thread thread)  查询指定的线程是否正在等待获取此锁定。
- lock.hasQueuedThreads() 的作用是查询是否有线程正在等待获取此锁定。
- boolean hasWaiters(Condition condition) 查询是否有线程正在等待与此锁定有关的condition条件。
- boolean isFair() 判断是不是公平锁
- boolean isHeldByCurrentThread()查询当前线程是否保持此锁
- boolean isLocked()的作用是查询此锁定是否由任意线程保持
- void lockInterruptibly 如果当前线程未被中断则获取锁定，如果已经被中断则出现异常。
- boolean tryLock()的作用，仅当在被调用时锁定未被另一个线程保持的情况下，才获取该锁定。
- boolean tryLock(long timeout，TimeUnit unit)的作用是，如果锁定在给定等待时间内没有被另一个线程保持，且当前线程未被中断，则获取该锁定。
- awaitUninterruptibly() 调用该方法的前提是，当前线程已经成功获得与该条件对象绑定的重入锁，否则调用该方法时会抛出IllegalMonitorStateException。调用该方法后，结束等待的唯一方法是其它线程调用该条件对象的signal()或signalALL()方法。等待过程中如果当前线程被中断，该方法仍然会继续等待，同时保留该线程的中断状态。


# ReentrantReadWriteLock的使用

类ReentrantLock 具有完全互斥排他的效果。同一时间只有一个线程在执行ReentrantLock.lock()方法后面的任务。保证了线程的安全性，但是效率非常低下。ReentrantReadWriteLock是一种读写锁。读写锁表示也有两个锁，一个是读操作相关的锁，称为共享锁；一个是和写操作相关的锁，也叫排他锁。也就是多个读锁之间不互斥，读写与写锁互斥，写锁与写锁互斥。





