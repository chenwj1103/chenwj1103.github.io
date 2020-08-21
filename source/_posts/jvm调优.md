---
title: jvm调优
copyright: true
date: 2020-06-10 15:56:39
tags: jvm
categories: jvm
---

#### 什么情况下需要调优？

>  JVM 内存分配不合理最直接的表现就是频繁的 GC，这会导致上下文切换等性能问题，从而降低系统的吞吐量、增加系统的响应时间。
>
>  因此，如果你在线上环境或性能测试时，发现频繁的 GC，且是正常的对象创建和回收，这个时候就需要考虑调整 JVM 内存分配了，从而减少 GC 所带来的性能开销。
>
>  - 调整晋升到老年代的晋升年龄；-XX:PetenureSizeThreshold
>
>  - 年轻带和老年代 –XX:NewRatio  1：2；
>
>  - Eden：survivor  -XX:SurvivorRatio：8
>
>  - 动态调整  -XX:+UseAdaptiveSizePolicy  
>
>  - 查询虚拟机默认的堆大小
>
>   ```
>   MacBook-Pro:conf zhuningning$ java -XX:+PrintFlagsFinal -version | grep HeapSize
>       uintx ErgoHeapSizeLimit                         = 0                                   {product}
>       uintx HeapSizePerGCThread                       = 87241520                            {product}
>       uintx InitialHeapSize                          := 268435456                           {product}
>       uintx LargePageHeapSizeThreshold                = 134217728                           {product}
>       uintx MaxHeapSize                              := 4294967296                          {product}
>   java version "1.8.0_161"
>   Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
>   Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)
>   ```
>
>  使用gc日志方便查看问题
>
>  ```
>  - -XX:PrintGCTimeStamps：打印 GC 具体时间；
>  - -XX:PrintGCDetails ：打印出 GC 详细日志；
>  - -Xloggc: path：GC 日志生成路径
>  ```
>
>  

