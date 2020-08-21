---
title: kafka源码剖析-server端
copyright: true
date: 2019-09-13 17:13:09
tags: kafka server
categories: kafka

---

![kafka的zookeeper存储结构](/images/kafka/server/kafka的zookeeper存储结构.png)

[zookeeper在kafka中的作用](https://www.cnblogs.com/huazai007/articles/10990449.html)

整理认识下kafka server端的架构图

![server](/images/kafka/server/server.png)

# 网络层


## reactor模式

- 工作原理

1.首先创建serverSocketChannel对象并在selector上注册op_accept事件，serverSocketChannel负责监听指定端口上的连接请求；
2.当客户端发起到服务端的网络连接时，服务端的selector监听到此op_accept事件，会触发selector来处理op_accept；
3.当acceptor接收到来自客户端的socket连接请求时会为这个连接创建响应的socketChannel，将socketChannel设计为非阻塞模式，并在selector上注册关注的IO事件，如OP_READ,OP_WRITE.此时客户端与服务端的socket连接正式完成。
4.当客户端通过上面建立的socket连接想服务端发送请求时，服务端的selector会监听到op_read事件，并触发执行响应的处理逻辑。当服务端向客户端写数据的时候，客户端的selector会监听到op_write事件，并处罚响应的处理逻辑。

注意治理的所有事件处理逻辑都是在同一个线程中完成的。

而kafka使用的事多线程 多个selector的设计实现的

![Nio-多线程模型](/images/kafka/server/Nio-多线程模型.png)

## socketServer

![socketServer](/images/kafka/server/socketServer.png)

![serverSocket核心字段](/images/kafka/server/serverSocket核心字段.png)

## acceptor

acceptor的主要功能是接收客户端建立连接的请求，创建socket连接并分配给processor处理。

acceptor.run方法是acceptor的核心逻辑，其中完成了对OP_ACCEPT时间的处理。

![acceptorRun方法](/images/kafka/server/acceptorRun方法.png)

## processor

processor主要用于完成读取请求和写回响应的操作，processor不处理具体业务逻辑

![processor主要参数](/images/kafka/server/processor主要参数.png)

processor.run()方法实现了从网络连接上读取数据的功能。

![processorRun方法](/images/kafka/server/processorRun方法.png)

## RequestChannel

![RequestChannel](/images/kafka/server/RequestChannel.png)

![processor处理请求](/images/kafka/server/processor处理请求.png)


# API层

kafka的handler线程会取出processor线程，放入requestChannel的请求进行处理，并将产生的响应通过requestChanel传递给processor线程。handler线程属于kafka的api层。

## kafkaRequestHandler

kafkaRequestHandler的主要职责是从RequestChannel获取请求并调用kafkaApis.handle()方法处理请求

## kafkaApis

其是kafka服务器处理请求的入口类，它负责将kafkaRequestHandler传递过来的请求分发到不同的handl*处理方法中。



````
/**
   * Top-level method that handles all requests and multiplexes to the right api
   */
  def handle(request: RequestChannel.Request) {
    try {
      trace("Handling request:%s from connection %s;securityProtocol:%s,principal:%s".
        format(request.requestDesc(true), request.connectionId, request.securityProtocol, request.session.principal))
      ApiKeys.forId(request.requestId) match {
        case ApiKeys.PRODUCE => handleProducerRequest(request)
        case ApiKeys.FETCH => handleFetchRequest(request)
        case ApiKeys.LIST_OFFSETS => handleOffsetRequest(request)
        case ApiKeys.METADATA => handleTopicMetadataRequest(request)
        case ApiKeys.LEADER_AND_ISR => handleLeaderAndIsrRequest(request)
        case ApiKeys.STOP_REPLICA => handleStopReplicaRequest(request)
        case ApiKeys.UPDATE_METADATA_KEY => handleUpdateMetadataRequest(request)
        case ApiKeys.CONTROLLED_SHUTDOWN_KEY => handleControlledShutdownRequest(request)
        case ApiKeys.OFFSET_COMMIT => handleOffsetCommitRequest(request)
        case ApiKeys.OFFSET_FETCH => handleOffsetFetchRequest(request)
        case ApiKeys.GROUP_COORDINATOR => handleGroupCoordinatorRequest(request)
        case ApiKeys.JOIN_GROUP => handleJoinGroupRequest(request)
        case ApiKeys.HEARTBEAT => handleHeartbeatRequest(request)
        case ApiKeys.LEAVE_GROUP => handleLeaveGroupRequest(request)
        case ApiKeys.SYNC_GROUP => handleSyncGroupRequest(request)
        case ApiKeys.DESCRIBE_GROUPS => handleDescribeGroupRequest(request)
        case ApiKeys.LIST_GROUPS => handleListGroupsRequest(request)
        case ApiKeys.SASL_HANDSHAKE => handleSaslHandshakeRequest(request)
        case ApiKeys.API_VERSIONS => handleApiVersionsRequest(request)
        case ApiKeys.CREATE_TOPICS => handleCreateTopicsRequest(request)
        case ApiKeys.DELETE_TOPICS => handleDeleteTopicsRequest(request)
        case requestId => throw new KafkaException("Unknown api code " + requestId)
      }
    } catch {
      case e: Throwable =>
        if (request.requestObj != null) {
          request.requestObj.handleError(e, requestChannel, request)
          error("Error when handling request %s".format(request.requestObj), e)
        } else {
          val response = request.body.getErrorResponse(e)

          /* If request doesn't have a default error response, we just close the connection.
             For example, when produce request has acks set to 0 */
          if (response == null)
            requestChannel.closeConnection(request.processor, request)
          else
            requestChannel.sendResponse(new Response(request, response))

          error("Error when handling request %s".format(request.body), e)
        }
    } finally
      request.apiLocalCompleteTimeMs = time.milliseconds
  }

````

# 日志处理

## 基本概念

kafka使用日志文件保存生产者发送的消息，每条消息使用offset值保存它在分区中的偏移量，offset是逻辑值。同一个分区中的消息是顺序写入的。

分区的任一副本都会有响应的log文件；

为了避免日志文件太大，在对应的磁盘上建一个目录，命名规则是topicName-partitionId,log与分区之间是一一对应的；

kafka将log文件通过分段的方式分成多个logSegment文件，logSegment是一个逻辑上的概念，一个LogSegment文件对应磁盘上的一个日志文件和一个索引文件，其中日志文件记录消息，索引文件保存了消息的索引。日志文件大小达到一个阈值是，就会创建新的文件。命令规则是[baseOffset].log, 是第一条消息的offset。

![Log的结构](/images/kafka/server/Log的结构.png)

为了提高查询消息的效率，索引文件不并没有为每条消息建立索引项，而是使用稀疏索引方式为文件中的部分消息简历了索引

![索引-日志文件](/images/kafka/server/索引-日志文件.png)

## fileMessageSet

fileMessageSet在磁盘上对应一个日志文件，它集成了MessageSet抽象类，它分为三部分：8个字节的offset值；4个字节的size表示messageData的大小；这两个部分组成LogOverhead,message data部分保存了消息的数据，逻辑上对应一个Message对象

![fileMessageSet对象](/images/kafka/server/fileMessageSet对象.png)

![Message类消息](/images/kafka/server/Message类消息.png)


## ByteBufferMessageSet

常见的算法是数据量越大压缩率越高；kafka使用的压缩方式是将多个消息一起进行压缩；服务端之间进行传输数据是压缩状态，而消费者从服务端拉取的数据也是压缩的。

## offsetIndex

![offsetIndex](/images/kafka/server/offsetIndex.png) 

## LogSegment

LogSegment中封装了一个FileMessageSet和一个offsetIndex对象，提供日志文件和索引文件的读写功能以及其他辅助功能。

![LogSegment](/images/kafka/server/LogSegment.png) 


### LogSegment的read方法

四个参数：

- startOffset:指定读取的其实消息的offset；
- maxOffset: 指定读取的结束的offset；
- maxSize: 指定读取的最大字节数；
- maxPosition: 指定读取的最大无力日志，默认是日志文件的大小；

读取日志文件之前需要将startOffset和maxOffset转化为对应的无力地址才能使用；

![LogSegmentRead方法](/images/kafka/server/LogSegmentRead方法.png) 

![Log使用跳表的流程](/images/kafka/server/Log使用跳表的流程.png) 


## Log

Log是对多个LogSegment对象的顺序组合，形成一个逻辑的日志，为了实现快读定位LogSegment，log使用跳表来对LogSegment进行管理；

jdk中有跳表的实现-concurrentSkipListMap，它是一个线程安全的实现；

在log中将每个LogSegment的baseOffset作为key，LogSegment对象作为value，放入segment这个跳表结构中

![Log使用跳表的流程](/images/kafka/server/Log使用跳表的流程.png) 

![logAppend方法大致流程](/images/kafka/server/logAppend方法大致流程.png) 

## LogManager

- 简介

在一个broker上的所有log都是由LogManager进行管理的，LogManager提供了加载Log、创建Log集合、删除Log集合、查询Log集合等功能，并且启动了3个周期性的后台任务以及多个线程分别是：log-flusher（日志刷写任务）、log-retention（日志保留）任务，检查点刷新任务以及cleaner线程（日志清理）；

![三个周期性任务的描述](/images/kafka/server/三个周期性任务的描述.png)

![日志压缩功能](/images/kafka/server/日志压缩功能.png)

![日志压缩-cleaner线程](/images/kafka/server/日志压缩-cleaner线程.png)

![LogManager初始化流程](/images/kafka/server/LogManager初始化流程.png)



# 副本机制

## 副本简介

![副本的概念](/images/kafka/server/副本的概念.png)

- 本地和远程副本

![本地和远程副本](/images/kafka/server/本地和远程副本.png)

![副本的更新操作](/images/kafka/server/副本的更新操作.png)

## 分区

服务端使用partition表示分区，partition负责管理每个副本对应的replica对象，进行leader副本的切换，isr集合的管理以及调用日志存储自行他完成写入消息

partition的核心字段及主要方法：

![partition的核心字段](/images/kafka/server/partition的核心字段.png)

### 创建副本

![创建副本](/images/kafka/server/创建副本.png)

### 副本将角色切换

broker会根据kafkaController发送的leaderAndISRRequest请求控制副本的leader和follower副本角色切换，Partition.makeLeader()是LeaderAndISRRequest中比较重要的环节之一。

![makerLeader中用到的参数](/images/kafka/server/makerLeader中用到的参数.png)

![makeLeader代码1](/images/kafka/server/makeLeader代码1.png)

![makeLeader代码2](/images/kafka/server/makeLeader代码2.png)

- 增加leader副本的HW

当isr集合发生增减或是ISR集合中任一副本的LEO发生变化时，都会导致ISR集合中最小的LEO变大。获取ISR集合中最小的LEO作为新的HW，比较现在的HW和新的HW，取较小的作为HW；


### ISR集合管理

partition除了对副本的leader和follower角色进行管理，还需要管理ISR集合。随着follower副本不断与leader副本进行消息同步，follower副本的leo会逐渐后移，并最终赶上leader副本的leo，最终该follower会进入ISR集合。

![添加ISR元素的实现](/images/kafka/server/添加ISR元素的实现.png)

![减少ISR元素的实现](/images/kafka/server/减少ISR元素的实现.png)

SIR集合发生增减的时候，都会将最新的ISR集合保存在zookeeper中。

### 追加消息

在分区中，只有leader副本能够处理读写请求，partition.appendMessagesToLeader()方法提供了向leader副本对应的Log中追加消息的功能。当isr集合中副本的数量小于配置的最小的限制，且生产者对可用性有较高的可用性，则不能支架消息，会产生一个NotEnoughReplicasException异常。

否则追加消息到对应的leader副本，尝试增加leader的HW；

### checkEnoughReplicasReachOffset

![checkEnoughReplicasReachOffset](/images/kafka/server/checkEnoughReplicasReachOffset.png)


## ReplicaManager 副本管理机制

![ReplicaManager1](/images/kafka/server/ReplicaManager1.png)

![ReplicaManager2](/images/kafka/server/ReplicaManager2.png)

### 副本角色切换

在kafka集群中会选择一个broker称为kafkaController的leader，他负责管理整个kafka集群。controller leader根据partition的leader副本和follower副本的状态向对应的broker节点发送leaderAndIsrRequest，整个请求主要用于副本的角色切换。

![leaderAndIsrRequestAndResponse](/images/kafka/server/leaderAndIsrRequestAndResponse.png)

![becomeLeaderAndFollower](/images/kafka/server/becomeLeaderAndFollower.png)

![makerFollower](/images/kafka/server/makerFollower.png)


updateFollowerLogReadResults()方法主要针对来自follower副本的fetchRequst多了异步处理。

- 更新leader副本上维护的follower副本的各项状态，入LEO等；
- 更新follower副本不断fetch的消息，最终追上leader副本，可能对ISR集合进行可扩张，同事将ISR集合的记录保存的zookeeper；
- 检测是否需要后移leader副本的HW；

### 消息同步

AbstractFetcherManager.addFetcherForPartitions()方法会让follower副本从指定的offset开始与leader副本进行同步。改方法的参数设计brokerAndInitialOffset类，他封装了broker的网络位置信息以及同步的其实offset。

![addFetcherForPartitions同步副本消息](/images/kafka/server/addFetcherForPartitions同步副本消息.png)

removeFetcherForPartitions()方法会停止指定follower副本的同步操作；如果fetcher线程不在为任何分区的follower副本提供同步，则会被shutdown掉。

![AddPartitionsAndRemovePartitions](/images/kafka/server/AddPartitionsAndRemovePartitions.png)

![副本请求的offset超过了leader的范围](/images/kafka/server/副本请求的offset超过了leader的范围.png)

### 关闭副本

![关闭副本](/images/kafka/server/关闭副本.png)

### replicaManager中的定时任务

![replicaManager中的定时任务](/images/kafka/server/replicaManager中的定时任务.png)

### metadataCache

metadataCache是broker用来缓存整个集群中全部分区状态的组件，kafkaController通过向集群中的broker发送updateMetadataRequest来更新其metadataCache组件中的缓存的数据。

![metadataCache的字段](/images/kafka/server/metadataCache的字段.png)


- getPartitionMetadata方法的实现

![getPartitionMetadata方法的实现1](/images/kafka/server/getPartitionMetadata方法的实现1.png)

![getPartitionMetadata方法的实现2](/images/kafka/server/getPartitionMetadata方法的实现2.png)







# groupCoordinator

每个broker上会实例化一个groupCoordinator对象，kafka按照consumer group名称将其分配给对应的groupCoordinator进行管理，每个groupCoordinator只负责管理consumer group的一个子集；

## groupCoordinator的功能

- 负责处理joinGroupRequest和syncGroupRequest完成对consumer group中分区的分配工作；
- 通过GroupMetadataManager和内部的topic维护offset信息，即使出现消费者宕机的情况，也可以找回之前提交的offset；
- 记录consumer group的相关信息，即使broker宕机导致consumer group 由新的groupCoordinator进行管理，新的groupCoordinator也可以知道consumer Group中的每个消费者负责处理那个分区等信息；
- 通过心跳检测消费者的状态；

![MemberMetadata的主要字段](/images/kafka/server/MemberMetada的主要字段.png)

![GroupMetadata元数据信息](/images/kafka/server/GroupMetadata元数据信息.png)

groupMetadata提供了对其字段的操作，包括对members集合的增删，对state的切换，同事需要选择group leader；

![选择groupLeader](/images/kafka/server/选择groupLeader.png)

## groupMetadataManager

groupMetadataManager是groupCoordinator中负责管理 consumer group元数据以及对应offset信息的组件，groupMetadataManager底层使用offsets topic，以消息的形式存储 consumer group的groupMetadata信息以及其消息的每个每个分区的信息

![GroupMetadataManager字段](/images/kafka/server/GroupMetadataManager字段.png)

- removeGroup方法的实现

![removeGroup方法的实现](/images/kafka/server/removeGroup方法的实现.png)

### 查找groupCoordinator

![查找groupCoordinator](/images/kafka/server/查找groupCoordinator.png)

![GroupCoordinator-offsetPartition-ConsumerGroup](/images/kafka/server/GroupCoordinator-offsetPartition-ConsumerGroup.png)

### loadGroupsAndOffsets 方法

![loadGroupsAndOffsets](/images/kafka/server/loadGroupsAndOffsets.png)

### SyncGroupRequest相关处理

consumer group中的leader消费者通过SyncGroupRequest将分区的分配结果发送给GroupCoordinator,GroupCoordinator会根据此分配结果形成SyncGroupResponse返回给所有的消费者。

### offsetFetchRequest和listGroupRequest处理

![offsetFetchRequest处理](/images/kafka/server/offsetFetchRequest处理.png)

## groupCoordinator分析

![groupCoordinator各个字段的含义](/images/kafka/server/groupCoordinator各个字段的含义.png)

- groupState字段的四个状态

![groupState的四个状态](/images/kafka/server/groupState的四个状态.png)

![groupState的四个状态转化](/images/kafka/server/groupState的四个状态转化.png)

- joinGroup()方法

![JoinGroup方法](/images/kafka/server/JoinGroup方法.png)

![dojoinGroup方法](/images/kafka/server/dojoinGroup方法.png)

![prepareRebalance方法](/images/kafka/server/prepareRebalance方法.png)

- doSyncGroup()方法

![doSyncGroup操作](/images/kafka/server/doSyncGroup操作.png)

- commitOffsetRequest()方法

![commitOffsetRequest](/images/kafka/server/commitOffsetRequest.png)

![commitOffsetRequest2](/images/kafka/server/commitOffsetRequest2.png)

- leaveGroupRequest

![leaveGroupRequest](/images/kafka/server/leaveGroupRequest.png)

# kafkaController

## kafkaController简介

![kafkaController简介](/images/kafka/server/kafkaController简介.png)

![broker在zookeeper中的存储信息](/images/kafka/server/broker在zookeeper中的存储信息.png)

![kafkaController组件](/images/kafka/server/kafkaController组件.png)

kafkaController是zookeeper与kafka集群交互的桥梁，他一方面对zookeeper进行监听，其中包括broker写入zookeeper中的数据，也包括管理员使用脚本写入的数据;另一方面根据zookeeper中数据的变化做出相应的处理，通过Request请求控制每个broker，kafkaControlle也通过zookeeper提高了高可用的机制。

## ControllerChannelManager

ControllerChannelManager主要管理broker之间的网络交互。controller只能发送leaderAndISRRequest、stopReplicaRequest、updateMetadataRequest三种请求；

ControllerChannelManager的核心字段是brokerStatInfo，主要用于管理集群中各个broker对应的brokerStatInfo对象。

![ControllerChannelManager管理brokerStateInfo](/images/kafka/server/ControllerChannelManager管理brokerStateInfo.png)

## controllerContext

controllerContext中维护了controller使用到的上下文信息，从其构造函数可以猜到controllerContext与zookeeper有密切的关系，可以看做两个之间的缓存


## controllerBrokerRequestBatch

为了提高broker leader与集群中其它broker的通信效率，kafka controller使用controllerBrokerRequestBatch实现批量发送请求的功能。

![controllerBrokerRequestBatch的核心字段](/images/kafka/server/controllerBrokerRequestBatch的核心字段.png)

## partitionStateMachine

partitionStateMachine是controller leader用于维护分区状态的状态机，分区的状态是通过partitionState接口定义的 

![partitionState及转换](/images/kafka/server/partitionState及转换.png)

![状态转换时完成的操作](/images/kafka/server/状态转换时完成的操作.png)


selectLeaderForPartition()方法

- 使用指定的partitionLeaderSelector()为分区选举新的leader副本；
- 将leader副本和isr集合的信息写入zookeeper;
- 更新context.partitionLeadershipInfo集合中缓存的leader副本、isr集合等信息；
- 将上述确定的leader副本，isr集合，ar集合等信息添加到controllerBrokerRequestBatch，之后会封装成leaderAndIsrRequest发送相关的broker；

## partitionLeaderSelector

offlinePartitionLeaderSelector会根据currentLeaderAndIsr选举新的leader和isr集合；

![offlinePartitionLeaderSelector](/images/kafka/server/offlinePartitionLeaderSelector.png)


- 选举的代码实现

![SelectLeader1](/images/kafka/server/SelectLeader1.png)

![SelectLeader2](/images/kafka/server/SelectLeader2.png)


## ReplicaStateMachine

ReplicaStateMachine是controller leader 用于维护副本状态的状态机，副本状态由replicaState接口表示

![replicaState状态](/images/kafka/server/replicaState状态.png)

![replicaState状态变化所做操作](/images/kafka/server/replicaState状态变化所做操作.png)

## zookeeperListener


I0Itec-zkClient 是zookeeper的客户端工具

### listener接口介绍

kafkaController 会通过zookeeper监控整个kafka集群的运行状态。具体实现是在zookeeper的指定节点添加listener，监听此节点中的数据变化或者其子节点的变化，从而触发响应的业务逻辑。

IZKDataListener监听指定节点的数据变化；IZKChildListener监听指定节点的子节点变化；IZKStateListener监听zookeeper连接状态的变化；

![kafka实现的zkListener接口](/images/kafka/server/kafka实现的zkListener接口.png)

![topicChangeListener实现](/images/kafka/server/topicChangeListener实现.png)

![deleteTopicChildChange](/images/kafka/server/deleteTopicChildChange.png)

![deleteTopicChildChange2](/images/kafka/server/deleteTopicChildChange2.png)

partitionModificationListener 主要用于监听一个topic分区的变化

![partitionModificationsListener](/images/kafka/server/partitionModificationsListener.png)

brokerChangeListener 主要负责处理broker的上线和故障下线，上线时会在“/brokers/ids”下创建临时节点，下线时会删除对应的临时节点。

![brokerChildChange](/images/kafka/server/brokerChildChange.png)

![onBrokerFailure](/images/kafka/server/onBrokerFailure.png)

![onBrokerFailureDemo1](/images/kafka/server/onBrokerFailureDemo1.png)

![onBrokerFailureDemo2](/images/kafka/server/onBrokerFailureDemo2.png)

副本重新分配的listener

partitionReassignedListener 监听zookeeper节点是 "/admin/reassign_partitions",当管理人员通过ReassignPartitionsCommond名指定某些分区需要重新分配副本时，会将指定分区的信息写入改节点，从而触发partitionReassignedListener

![副本重新分配的步骤1](/images/kafka/server/副本重新分配的步骤1.png)

![副本重新分配的步骤2](/images/kafka/server/副本重新分配的步骤2.png)

![副本重新分配的步骤3](/images/kafka/server/副本重新分配的步骤3.png)



## kafkaController初始化与故障转移

kafkaController的启动和故障转移的过程与zookeeperLeaderElector有着密切的关系，

zookeeperLeaderElector中有两个比较重要的字段，leaderId以及leaderChangeListener

![zookeeperLeaderElector](/images/kafka/server/zookeeperLeaderElector.png)


### 触发选举

- 第一次启动的时候；
- leaderChangeListener监听到"/controller"节点中的数据被删除；
- zookeeper连接过期并重新连接之后；

![elect](/images/kafka/server/elect.png)

### onControllerFailover

elect方法中调用onBecomingLeader()方法实际上还是onControllerFailover方法。当选举成功后，会完成一系列初始化操作。

![onControllerFailover1](/images/kafka/server/onControllerFailover1.png)

![onControllerFailover2](/images/kafka/server/onControllerFailover2.png)

### ControllerContext 从zookeeper中获取的信息

![ControllerContext从zookeeper中获取信息](/images/kafka/server/ControllerContext从zookeeper中获取信息.png)

### partition rebalance

![优先副本](/images/kafka/server/优先副本.png)

### 优先选举 checkAndTriggerPartitionRebalance

![优先选举1](/images/kafka/server/优先选举1.png)

![优先选举2](/images/kafka/server/优先选举2.png)

## 处理controlledShutDownRequest

更换硬件、系统升级可能需要管理broker，kafka提供了方法来管理,主动下线broker

![controlledShutDownRequest](/images/kafka/server/controlledShutDownRequest.png)









































































































































































































