---
title: kafka源码剖析-快速入门
copyright: true
date: 2018-04-05 15:56:51
tags: kafka源码剖析
categories: kafka

---

# 简介

kafka是一种分布式的,基于发布订阅的消息系统.由scala语言编写的.由linkedin于2011年开源,2012年10月从apache孵化器毕业的顶级项目.

## 主要特性

1. **kafka具有近乎实时的消息处理能力**,面对海量消息的查询和存储也能高效的处理.kafka消息存储在磁盘中,它以顺序读写的方式访问磁盘,从而避免了随机读写磁盘导致的性能瓶颈.
2. **kafka支持批量读写消息,并且会对消息进行批量压缩**.这样提高了网络的利用率,也提高了压缩效率.
3. **kafka支持消息分区,每个分区中的消息可以保证顺序传输,而分区之间的消息可以并发操作,这样提高了kafka的并发能力**.
4. kafka支持在线增加分区,支持在线水平扩展.
5. kafka支持为多个分区创建多个副本,其中只会有一个leader副本负责读写,其它副本负责与leader进行同步,提高了数据的容灾能力,kafka将会将leader副本均匀的分布在集群中的进服务器上,实现性能的最大化.

## kafka的应用场景

1. kafka可以作为传统的消息中间件,实现消息队列和消息的发布订阅.
2. kafka可以作为系统中的数据总线,将其接入多个子系统,子系统可以将消息推送到kafka中进行保存,之后流转到目的的系统中.
3. kafka可以作为日志收集中心,多个系统产生的日志统一收集到kafka中,然后由数据分析平台进行统一管理.日志会被kafka持久化到硬盘,同时支持离线数据处理和实时数据处理.
4. kafka可以作为数据库主从同步的工具.


## 以kafka为中心的解决方案

kafka的作用

1. **解耦合**,开发人员不需要知道各个子系统/服务/存储之间的关系,只需要面向kafka编程即可.两者只需要知道消息存放的topic和消息中的数据的格式即可.简单说,一个扮演生生产者一个扮演消费者,kafka是消息队列.
2. **数据持久化**.网络传输时不可靠的,kafka把数据以消息的形式持久化到磁盘.即使kafka出现宕机,数据也不会丢失.kafka还提供了**日志清理和日志压缩**等功能.另外在磁盘操作中,耗时最长的是寻址时间,kafka采用**顺序读写**的方式,实现了高吞吐.
3. **扩展和容灾** kafka的每个topic可以分为多个partition(分区),每个分区有多个replica(副本),实现了消息的冗余备份.各个分区中的消息是不同的,类似于数据库水平切分的思想.提高了并发读写能力.同一分区的不同副本之间存储的是相同的消息,副本之间是一主多存的关系,其中leader负责消息的读写,follower负责与leader进行备份,如果leader出现故障,follower重新选取leader副本提供服务.这样,通过分区的数量实现水平扩展,通过副本的数量提高容灾能力.
   
   而且kafka的consumer端采用的是pull拉取消息,consumer端保存消息消费的具体位置,如果宕机重启后,consumer自己决定何时从哪消费消息;
   
   kafka的consumer水平扩展,可以让多个consumer加入一个consumer组.在一个consumer group中,每个分区只能分配给一个consumer消费,当kafka服务端增加分区后,consumer group中可以添加consumer来提高consumer group的消费能力,当consumer group中的某个consumer出现故障时,会通过rebalance操作,将下线consumer消费的分区分配给其它consumer进行消费.当然一个consumer group可以订阅多个topic,每个consumer可以同时处理多个分区.
   
4. **顺序保证**,kafka保证一个partition中的消息的有序性,不保证多个partition之间的数据有序性.
5. **缓冲和峰值处理能力**,kafka的吞吐量较大,kafka能够顶住突发的访问压力,不会因为突发的压力造成系统崩溃不可用.
6. **异步通信** ,提高处理其它的能力.

# kafka的核心概念

## 消息

消息是kafka最基本的数据单位,消息是由一串字节构成,其中主要由key和value组成,其中key和value都是byte数组.key的主要作用时根据一定的策略将消息路由到指定的同一分区中,保证消息的顺序.

## topic/分区(partition)/log

topic是用于存储消息的逻辑概念,是一个消息集合.每个topic可以划分成多个分区,同一topic下的不同分区包含的信息是不同的.每个消息在被添加到分区中会被分配一个offset,它时消息在该分区中的唯一编号,kafka通过offset保证消息在分区内的顺序,同一分区中的消息是有序的,不同的分区内的消息,kafka不能保证其顺序性.

分区是kafka水平扩展的基础,增加kafka的分区可以增加kafka的并行处理能力.

分区在逻辑上对应一个log,当消息写入分区时,实际上是写入到了分区对应的log中.log是一个逻辑概念,对应磁盘上的一个文件夹,log由多个segment组成,每个segment对饮一个日志文件和索引文件.每一个日志文件是有大小限制的.索引文件采用稀疏索引的方式,大小并不会很大,在运行时将其映射到内存,提高索引速度.

kafka的topic在每个机器上是以文件存储的,而这些文件呢,会分目录,partition就是文件的目录.

## 保留策略和日志压缩

kafka中有两种保留策略,一种时根据消息保留的时间,一种时根据topic存储消息的大小.kafka有一个线程定时检查消息的大小.

此外kafka还会进行日志压缩,消息的key与value对应的值时不断变化的,消费者只关心最新value值,开启kafka的日志压缩功能,定期将相同可以的消息进行合并,只留最新的value的值.


## broker

一个kafka服务器就是一个broker,broker的主要工作就是接收生产者的消息,分配offset,保存到硬盘.接收消费者和其它broker的请求,并根据请求类型处理并返回相应.

## 副本(replica)

kafka对消息进行冗余备份,每个partition有多个副本,每个副本中的消息有多个副本.选出一个leader副本负责读写请求.follower副本只是进行冗余备份和容错.

## ISR (In-Sync Replica) 

ISR表示目前可用且与leader相差不多的副本集合,在broker宕机后重新选举新的leader继续对外提供服务.ISR集合必须满足以下条件

1. 副本所在的节点必须维持着与zookeeper的联系
2. 副本最后一条消息的offset与leader副本的最有一条消息的offset的差值不能超过指定值.

某个副本可能由于某种原因从ISR集合中退出,在满足以上条件后,会重新加入ISR集合.

## HW/LEO

highWaterMark标记了一个特殊的offset,当消费者消费消息的时候,只能拉取到HW之前的消息,HW之后的消息对消费者来说时不可见的,HW也是由LEADER副本管理,当ISR集合中全部非follower副本都拉取HW制定消息进行同步后,leader副本会增加hw的值.

log end offset 是所有的副本都会标记的一个offset,它指向最后追加到当前副本的最后一个消息,当生产者向leader追加消息的时候,leader副本的leo标记会增加;当follower副本成功从leader副本中拉取消息更新到本地的时候,foller副本的leo就会增加,当isr集合中的所有的副本都完成了该消息的leo增加,则leader副本会增加HW.

## 副本的同步复制

同步复制要求所有的能工作的follower副本完成复制,这条消息才会被认为提交成功.一旦有一个follower副本出现故障,就导致HW无法完成递增,消息就无法提交,生产者拿不到消息,这样故障的follower副本会拖慢整个体系的性能.

## 副本的异步复制

异步复制中,leader副本收到生产者推送的消息后,就会被认为该消息提交成功,follower副本异步的从leader中同步消息.这样设计虽然避免了同步复制的问题,但同样存在一定的风险.

假设follower副本同步的消息比较慢,它保存的消息圆圆落后于leader副本,此时leader副本突然宕机进行选举的时候,副本会丢失消息.

此时ISR集合策略解决了这种问题. 当follower复制消息较慢的时候,副本被踢出isr集合.follower副本在更新消息的时候时批量写磁盘.

## Cluster/controller

多个broker可以做成一个cluster对外提供服务,每个cluster当中会选举出一个broker担任controller,controller是kafka集群的指挥中心,其它broker听从controller指挥实现功能,

controller负责管理分区的状态,管理每个分区的副本状态/监听zookeeper中的数据变化等工作,controller也是一主多从的实现.

## 生产者(producer)

生产者的主要工作时生产消息,并将消息安装一定的规则推送到topic的分区中.例如根据消息的key的hash值选择分区或按照轮训全部分区的方式.

ACK，是指服务器收到消息之后，是存下来之后，再给客户端返回，还是直接返回

````
该参数是设置的,request.required.acks

//0: 不等服务器ack就返回了，性能最高，可能丢数据 
//1. leader确认消息存下来了，再返回 
//all: leader和当前ISR中所有replica都确认消息存下来了，再返回（这种方式最可靠）

````
### 同步发送和异步发送

````
所谓异步发送，就是指客户端有个本地缓冲区，消息先存放到本地缓冲区，然后有后台线程来发送。

在异步发送下，有以下4个参数需要配置：
（1）队列的最大长度 
buffer.memory //缺省为33554432, 即32M

（2）队列满了，客户端是阻塞，还是抛异常出来（缺省是true) 
block.on.buffer.full 
//true: 阻塞消息 
//false：抛异常

（3）发送的时候，可以批量发送的数据量 
batch.size //缺省16384字节，即16K

（4）最长等多长时间，批量发送 
linger.ms //缺省是0 
//类似TCP/IP协议中的linger algorithm，> 0 表示发送的请求，会在队列中积攥，然后批量发送。

很显然，异步发送可以提高发送的性能，但一旦客户端挂了，就可能丢数据。
对于RabbitMQ, ActiveMQ，他们都强调可靠性，因此不允许非ACK的发送，也没有异步发送模式。Kafka提供了这个灵活性，允许使用者在性能与可靠性之间做权衡。

（5）消息的最大长度 
max.request.size //缺省是1048576，即1M

这个参数会影响batch的大小，如果单个消息的大小 > batch的最大值(16k)，那么batch会相应的增大

````


## 消费者(consumer)

消费者的主要工作时从topic中拉取消息,对消息进行消费.每个消费者消费哪个分区,消费到哪个位置都是有consumer自己维护的,这样减少了server端的开销,同时减少服务端宕机带来的风险.

消费者采用pull的方式进行消费,这样消费者可以根据自己的消费能力进行处理.

在kafka里面，是保证消息不漏，也就是at least once。至于重复消费问题，需要业务自己去保证，比如业务加判重表。



## 消费者组(consumer group)

在kafka中,多个consumer组成一个consumer group ,一个consumer只能属于一个consumer group.consumer group保证其订阅的topic的每个分区只能被分配给此consumer group中的一个消费者处理.如果不同的consumer group 订阅了同一个topic ,各个消费者组之间彼此不会干扰.

kafka消费者组是逻辑上的订阅者.kafka还通过消费者组实现了水平扩展和故障转移.

当消费者数量超过分区的数量时,会造成消费者分不到分区,从而造成消费者的浪费.

## kafka集群的架构

![kafka集群的架构](/images/kafka/introduce/kafka集群的架构.png)

生产者会根据业务逻辑生产消息,之后根据路由规则将消息发送到制定分区的leader副本所在的broker上,在kafka的服务端接收到消息后,会将消息追加到log中保存,之后follower副本会与leader副本进行同步,当isr集合中所有副本都完成了消息的同步后,则leader副本的hw会增加,并向生产者返回相应,

当消费者加入到consumer group时,会出发rebalance操作将分区分配给不同的消费者消费.随后,消费者会会恢复其消费位置,并向kafka服务端发送拉取的请求,leader副本会验证请求的offset及其它信息,最后返回.

# 搭建kafka源码环境

1. [scala环境](https://blog.csdn.net/he582754810/article/details/53837142)

2. [安装grandle](https://blog.csdn.net/hejjiiee/article/details/53510209)

3. [安装zookeeper环境](https://blog.csdn.net/Yan_Chou/article/details/53322429)

4. 源码构建

在kafka源码目录下,执行gradle idea命令

遇到问题 [源码构建报错](https://www.cnblogs.com/jun1019/p/7440468.html)

5. 安装scala插件

在idea中安装scala插件

6. 配置启动kafka

在kafka服务端使用log4j输出日志,启动前需要把config下的 log4j.properties 配置文件放到core的/scala/main/scala 下,然后运行程序

7. kafka启动参数配置

![kakfa启动参数配置](/images/kafka/introduce/kakfa启动参数配置.png)
































