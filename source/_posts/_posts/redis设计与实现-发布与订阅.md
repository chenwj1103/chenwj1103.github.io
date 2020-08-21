---
title: redis设计与实现-发布与订阅
copyright: true
date: 2018-01-21 00:36:05
tags: redis-发布与订阅
categories: redis
---

redis的订阅与发布功能由publish subscribe psubscribe等命令组成.客户端可以订阅**多个频道**或者**订阅多个模式**.从而成为这些明道的订阅者.当其它客户端向这些频道发送消息,这些地订阅者可以收到.


![将消息发送到频道订阅者和匹配模式的订阅者](/images/redis/subscribe/将消息发送到频道订阅者和匹配模式的订阅者.png)

订阅一个news.it频道和以及接收到其它客户端向这个频道发送来的信息.

````
订阅者
127.0.0.1:6379> subscribe "news.it"
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news.it"
3) (integer) 1

1) "message"
2) "news.it"
3) "hello"
1) "message"
2) "news.it"
3) "test"

发布者
127.0.0.1:6379> publish "news.it" hello
(integer) 1
127.0.0.1:6379> publish "news.it" test
(integer) 1

````

# 频道的订阅与退订

当一个客户端执行 SUBSCRIBE 命令， 订阅某个或某些频道的时候， 这个客户端与被订阅频道之间就建立起了一种订阅关系。

Redis 将所有频道的订阅关系都保存在服务器状态的 pubsub_channels 字典里面， 这个字典的键是某个被订阅的频道， 而键的值则是一个链表， 链表里面记录了所有订阅这个频道的客户端：

````
struct redisServer {

    // ...

    // 保存所有频道的订阅关系
    dict *pubsub_channels;

    // ...

}

````

一个pubsub_channels字典示例:

![一个pubsub_channels字典示例](/images/redis/subscribe/一个pubsub_channels字典示例.png)

## 订阅频道

每当客户端执行 SUBSCRIBE 命令， 订阅某个或某些频道的时候， 服务器都会将客户端与被订阅的频道在 pubsub_channels 字典中进行关联。

1. 如果频道已经有其他订阅者， 那么它在 pubsub_channels 字典中必然有相应的订阅者链表， 程序唯一要做的就是将客户端添加到订阅者链表的末尾。
2. 如果频道还未有任何订阅者， 那么它必然不存在于 pubsub_channels 字典， 程序首先要在 pubsub_channels 字典中为频道创建一个键， 并将这个键的值设置为空链表， 然后再将客户端添加到链表， 成为链表的第一个元素。

SUBSCRIBE 命令的实现可以用以下伪代码来描述：

````
def subscribe(*all_input_channels):

    # 遍历输入的所有频道
    for channel in all_input_channels:

        # 如果 channel 不存在于 pubsub_channels 字典（没有任何订阅者）
        # 那么在字典中添加 channel 键，并设置它的值为空链表
        if channel not in server.pubsub_channels:
            server.pubsub_channels[channel] = []

        # 将订阅者添加到频道所对应的链表的末尾
        server.pubsub_channels[channel].append(client)

````

## 退订频道

UNSUBSCRIBE 命令的行为和 SUBSCRIBE 命令的行为正好相反 —— 当一个客户端退订某个或某些频道的时候， 服务器将从 pubsub_channels 中解除客户端与被退订频道之间的关联：


1. 程序会根据被退订频道的名字， 在 pubsub_channels 字典中找到频道对应的订阅者链表， 然后从订阅者链表中删除退订客户端的信息。
3. 如果删除退订客户端之后， 频道的订阅者链表变成了空链表， 那么说明这个频道已经没有任何订阅者了， 程序将从 pubsub_channels 字典中删除频道对应的键。

# 模式的订阅与退订

与频道类似,服务器将所有模式的订阅关系都保存在服务器状态的pubsub_pattterns属性里面:

````
struct redisServer {

    // ...

    // 保存所有模式的订阅关系
    list *pubsub_pattterns;

    // ...

}

````
pubsub_pattterns是一个链表,链表中每个及诶点都包含着pubsub_patttern结构,这个结构的pattern属性记录了被订阅的模式,而client记录了订阅模式的客户端.

![pubsub_pattterns链表的实例](/images/redis/subscribe/pubsub_pattterns链表的实例.png)

## 订阅模式

每个客户端执行psubscribe命令的订阅某个模式的时候,服务器会对每个订阅的模式执行以下两个操作.

1. 新建一个pubsub_patttern的结构,将新结构的pattern属性设置为被订阅的模式,clients属性升值而为订阅模式的客户端.
2. 将pubsub_patttern结构添加到pubsub_pattterns链表的尾端.

执行psubscribe "news.*"

![执行psubscribe后的链表](/images/redis/subscribe/执行psubscribe后的链表.png)

## 退订模式

当punsubscribe执行的时候,服务器将遍历pubsub_pattterns链表,查找并删除那些pattern模式为被退订模式.


# 发送消息

当一个redis客户端执行publish命令将小心message发送到频道channel的时候,服务器执行以下2个动作:
1. 将message发送给channel频道的订阅者;
2. 如果有一个或者多个模式pattern与频道channel相匹配,那么消息message发送给pattern模式的订阅者;

## 将消息发送给频道的订阅者

将消息发送给channel频道的所有订阅者,就是将pubsub_chanels字典了找到频道的订阅者名单,然后将消息发送给其客户端.

![发送消息还给频道订阅者的伪代码实现](/images/redis/subscribe/发送消息还给频道订阅者的伪代码实现.png)

## 将消息发送给模式订阅者

将消息发送给channel频道模式相匹配的所有订阅者,就是将pubsub_patterns链表了找到频道相匹配的模式,然后将消息发送给其客户端.

![将消息发送给模式订阅者的伪代码实现](/images/redis/subscribe/将消息发送给模式订阅者的伪代码实现.png)


# 查看订阅信息

通过pubsub命令查看频道和模式的相关信息.

## pubsub channels [pattern]

返回服务器当前被订阅的频道,其中pattern是可选的参数;

![pubsub_channel伪代码实现](/images/redis/subscribe/pubsub_channel伪代码实现.png)

例子:

````
127.0.0.1:6379> pubsub channels
1) "news.it"

````

## pubsub numsub [chanel-1,channel-2...]

接收任意多个频道作为输入参数,并返回这些频道的订阅数量

![pubsub_numsub伪代码实现](/images/redis/subscribe/pubsub_numsub伪代码实现.png)

````
127.0.0.1:6379> pubsub numsub "news.it"
1) "news.it"
2) (integer) 1

````

## PUBSUB NUMPAT

返回服务器当前被订阅模式的数量,其实就是返回pubsub_patterns的链表的长度.


````
127.0.0.1:6379> pubsub numpat
(integer) 0

````


























