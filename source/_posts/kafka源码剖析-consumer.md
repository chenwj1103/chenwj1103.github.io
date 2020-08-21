---
title: kafka源码剖析-consumer
copyright: true
date: 2019-09-12 00:59:42
tags: kafka consumer
categories: kafka

---


# consumer简介

kafkaConsumer提供了一套封装良好的api，开发人员可以使用api轻松实现从服务端拉取消息，开发人员不必关系与kafka服务端的交互，比如网络管理、心跳检测、请求超时重试等底层操作，也不需要关心订阅topic的分区数量、分区leader副本的网络拓扑以及rebalance等操作，
而且kafka还有自动commit offset的功能；


# 传递保证语义

旧版本消费者的进度是记录在zookeeper中，为了缓解其压力，在服务端有一个名为 _consumer_offsets 的内部topic中记录消费进度，当出现rebalance操作时，对分区重新分配，rebalance操作完成后，消费者就可以重新读取offsets topic中的记录的offset，并从此位置继续消费。


consumer的poll方法返回的是最后一个消息的offset，为了避免消息丢失，建议处理完之后再poll消息，当然也可以手动commit  commitSync()以及commitAsync() 方法。

## 消息的传递保证有三个级别

- 至多一次  at most once:可能会丢失，绝不会重复传递；
- 至少一次  at least once ：消息绝不会丢失，可能会重复传递；
- 恰好一次 exactly once：每条消息只会被传递一次

至多一次一般不会出现、如果kafka保证传递的幂等性则使用至少一次也是没有问题的、恰好一次首先要保证不会产生重复的消息，其次消费者不能重复拉取相同的消息；

### 恰好一次-生产者

网络抖动可能出现至少一次的情况，所以为了实现恰好一次有以下两种方案：

- 每个分区只有一个生产者写入消息，当出现异常或者超时的情况时，生产者要查询此分区的最后一个消息，用来决定后续操作时消息重传还是继续；
- 为每个消息添加一个全局唯一主键，生产者不做特殊处理，按照之前的方式进行重传，保证至少一次，有消费者去去重；

###  恰好一次-消费者

拉取消息消费完之后，提交offset前出现宕机，这样重启后还会处理刚才那部分消息；拉取消息后先提交offset，宕机导致处理失败，则导致已提交的部分未做处理； 为了实现恰好一次有以下方案

- 消费者关闭自动提交offset的功能，且不再手动提交offset，这样就不适用offsets topic这个内部topic来记录其offset，而是有消费者自己保存offset在db中，用使用的原子性来实现确切一次的功能。消费者可以使用consumer.seek()方法手动设置消费位置，从此offset处开始继续消费；

### rebalance 操作

我们不知道rebalance操作以及那个分区分配给了哪个消费者，我们可以通过向consumer添加consumerReBalanceListener接口来解决这个问题：

- onPartitionsRevoked()方法：调用时机是consumer停止拉取数据之后。rebalance之前，我们可以在此方法中实现手动提交offset，这就避免了rebalance导致的重复消费的问题；
- onPartitionAssigned()方法：调用时机是rebalance完成之后，consumer开始拉取数据之前，我们可以在此方法中调整或者自定义offset值。


通过使用consumerReBalanceListener接口和seek()方法，我们就可以从关系型数据库中获取offset并手动设置了。


# consumer group rebalance 设计

最开始的时候交由zookeeper管理，严重依赖zookeeper，而且会产生羊群效应，脑裂问题；

之后交由broker管理，造成服务端的压力过大，而且要求服务端实现分配partition的方法，如果需要重新实现分配partition的策略，则需要修改服务端代码；

最后优化为交给消费者管理，0.9的版本进行了重新设计；

- 具体的策略

当消费者查找到管理当前consumer group的groupCoordinator后，就会进入join group阶段，consumer首先会向groupCoordinator发送joinGroupRequest请求，其中包含消费者的相关信息；

服务端的groupCoordinator收到joinGroupRequest后会暂存消息，收集到全部的消费者后，会根据joinGroupRequest中的信息来确定consumer group中的可用的消费者，从中选取一个消费者成为group leader，还会选取使用的分区分配策略，最后将这些消息封装成joinGroupResponse返回给消费者；

虽然每个消费者都会收到joinGroupResponse,但是只有group leader收到的joinGroupResponse中封装了所有的消费者信息，当消费者确定自己是group leader后，会根据消费者的信息以及选定的分区策略来进行分区分配；

在synchronizing group state阶段，每个消费者会发送syncGroupRequest到groupCoordinator，但是只有group leader的syncGroupRequest请求中保函了分区的分配结果， groupCoordinator根据group leader的分区结果形成syncGroupResponse返回给所有的consumer。


# kafkaConsumer分析

kafkaConsumer对外暴露了多个api，它是线程不安全的，可以使用线程池，线程池中的每个线程拥有一个kafkaProducer实例

- subscribe():订阅指定的topic，并为消费者自动分配分区；
- assign():用户手动订阅指定的topic，并指定消费者的分区；
- commit(): 提交消费者其实消费的位置；
- seek()：指定消费者的起始位置；
- poll(): 负责从服务端获取消息；
- pause() /resume()方法，暂停或者继续consumer，暂定后poll()方法会返回空；

## SubScribeState

![SubScribeState](/images/kafka/consumer/SubScribeState.png)


# rebalance操作

## 哪种情况下会出发rebalance

1. 有新的消费者加入consumerGroup
2. 有新的消费者宕机下线
3. 有消费者主动退出consumerGroup
4. consumerGroup订阅任一topic出现分区数量的变化
5. 消费者取消对topic的订阅

### 第一阶段

- 第一步检查是否需要重新查找groupCoordinator，主要是检查Coordinator字段是否为空以及与groupCoordinator之间的连接是正常；
- 第二步查找集群负载最低的node节点，并创建groupCoordinatorRequest,调用client.sent()方法将请求放入unsent队列中等待发送，并返回future对象，返回的对象经过compose方法适配，返回给heartbeatCompletionHandler;
- 第三步调用consumerNetClient.poll(future) 方法，groupCoordinatorRequest请求发送出去，此处使用阻塞的方式发送，知道收到groupCoordinatorResponse响应或异常完成，
- 第四步检查返回的RequestFuture<Void> 对象，如果出现retriableException异常，则调用ConsumerNetWorkClient.awaitMetadataUpdate()方法阻塞更新metadata中记录的集群元素后跳转到步骤一继续操作。如果不是RetriableException则直接报错；
- 第五步如果找到GroupCoordinator节点，但是网络连接失败，则将其unsent中对应的请求秦孔，并将coordinator字段置为空，重新查找GroupCoordinator

### 第二阶段

在成功查找到GroupCoordinator之后进入Join group阶段，在此阶段消费者会向GroupCoordinator发送joinGroupRequest请求，并处理响应

![joinGroupRequest](/images/kafka/consumer/joinGroup.png)

在进行完joinGroupRequest之后要进行joinGroupResponse()方法，

![joinGroupResponse](/images/kafka/consumer/joinGroupResponse.png)


### 第三阶段

在完成分区分配后，要进入synchronizing group state阶段，主要逻辑是向groupCoordinator发送syncGroupRequest，并处理syncGroupResponse响应。

![onjoinComplete](/images/kafka/consumer/onjoinComplete.png)

# offset 

offset commit 分为同步和异步提交以及手动提交，手动提交就是调用异步提交。

![offsetCommit](/images/kafka/consumer/offsetCommit.png)


## fetch offset

在rebalance结束后，每个消费者都确定了其需要消费的分区，在开始消费之前，消费者需要确定拉取消息的其实位置，假设之前已经将最后的消费者提交到了groupCoordinator中，groupCoordinator将其提交到内部的offset_topic中.此时消费者需要通过offsetFetchRequest 请求上次提交的位置，从此继续消费。

# fetcher

fetcher类的功能发送请求获取指定的消息集合，并更新消费位置。而且需要拉取最新的postion，有earlist、latest两种策略，上面两种策略都会发送offsetsRequest，请求指定的offset。


















