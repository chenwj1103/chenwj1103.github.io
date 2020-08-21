---
title: elasticsearch入门教程
date: 2018-01-18 15:56:37
tags: elasticsearch
categories: elasticsearch

---

# 需求分析

有一张答题记录表，主要字段如下。由于数据量比较大，所以按照学员id的后两位横向分表为100张表。

````
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `stu_id` int(11) NOT NULL COMMENT '学员id',
  `stu_total_score` decimal(4,1) DEFAULT NULL COMMENT '学员得分',, 
  `t_paper_id` int(11) NOT NULL DEFAULT '0' COMMENT '试卷id',
  `total_time` int(11) DEFAULT NULL COMMENT '答题总用时，单位为s',
  
````

有一需求，需要实时的根据stu_id获取该学员的paperId下的排名，以及学员排行榜（前10名），规则是按照分数降序、答题时间升序。

方案一：用union all 关键字关联100张表（是按照stu_id分表的），进行排序。然后获取学员的名次和排行榜。（由于数据量巨大，这种方案显然是不可取的）

方案二：使用redis的sorted set类型存储数据，但是细想其实 sorted set只能以一个字段作为排序字段，也是不可取的

方案三: 使用es，将所有的数据入es。

后续在生产环境使用es，由于有实时的查询排行榜的需求，所以在index数据入es的时候，实时的setRefresh操作，这样，在并发量比较大的时候，造成es响应很慢，大概50至60秒。

后来翻阅相关资料，发现es其实根本不适合做排序相关的排行榜需求，我们可以使用redis，将排序的2个字段加权后作为sorted set的score，实时证明，redis才是非常适合做实时排序相关功能的。


# 基本概念

## 近实时： 

Elasticsearch 是一个接近实时的搜索平台。这意味着，从索引一个文档直到这个文档能够被搜索到有一个很小的延迟（通常是 1 秒）。

## 集群（cluster）： 

一个集群就是由一个或多个节点组织在一起， 它们共同持有你全部的数据， 并一起**提供索引和搜索功能**。 一个集群由一个唯一的名字标识， 这个名字默认就是“elasticsearch”。 这个名字很重要， 因为一个节点只能通过指定某个集群的名字，来加入这个集群。一个集群中只包含一个节点是合法的。另外，你也可以拥有多个集群，集群以名字区分。

## 节点（node）：

一个节点是你集群中的一个服务器，作为集群的一部分，它存储你的数据，参与集群的索引和搜索功能。 和集群类似， 一个节点也是由一个名字来标识的， 默认情况下， 这个名字是一个随机的Marvel角色的名字，这个名字会在节点启动时分配给它。一个节点可以通过配置集群名称的方式来加入一个指定的集群。 默认情况下，每个节点都会被安排加入到一个叫做“elasticsearch”的集群中，这意味着，如果你在你的网络中启动了若干个节点， 并假定它们能够相互发现彼此，它们将会自动地形成并加入到一个叫做“elasticsearch” 的集群中。

 **注意：**如果当前你的网络中没有运行任何Elasticsearch节点，这时启动一个节点，会默认创建并加入一个叫做“elasticsearch”的单节点集群。
 
## 索引（index）: 

一个索引就是一个拥有相似特征的文档的集合。(相当于mysql的db) 。一个索引由一个名字来 标识（必须全部是小写字母的）

## 类型（type）：

在一个索引中，你可以定义一种或多种类型。（相当于mysql的table）一个类型是你的索引的一个逻辑上的分类/分区，其语义完全由你来定。通常，会为具有一组相同字段的文档定义一个类型。

## 文档（document）：

一个文档是一个可被索引的基础信息单元。（相当于mysql的一条数据）文档以JSON格式来表示

## 分片和复制（shards and replicas）：

### 分片 

一个索引可以存储超出单个结点硬件限制的大量数据。比如，一个具有10亿文档的索引占据1TB的磁盘空间，而任一节点可能**没有这样大的磁盘空间**来存储或者**单个节点处理搜索请求，响应会太慢**。

1. 允许你水平分割/扩展你的内容容量
2. 允许你在分片（位于多个节点上）之上进行分布式的、并行的操作，进而提高性能/吞吐量

当你创建一个索引的时候，你可以指定你想要的分片的数量。

### 复制

1. 在分片/节点失败的情况下，复制提供了高可用性。复制分片不与原/主要分片置于同一节点上是非常重要的。
2. 因为搜索可以在所有的复制上并行运行，复制可以扩展你的搜索量/吞吐量

分片和复制的数量可以在索引创建的时候指定。在索引创建之后，你可以在任何时候动态地改变复制的数量，但是你不能再改变分片的数量。

默认情况下，Elasticsearch中的每个索引分配5个主分片和1个复制。这意味着，如果你的集群中至少有两个节点，你的索引将会有5个主分片和另外5个复制分片（1个完全拷贝），这样每个索引总共就有10个分片。


# 安装 （待补充）

需要jdk1.7 +  

[下载 tar包](https://www.elastic.co/downloads/elasticsearch)

````
tar -xvf *.tar.gz
cd bin
./elasticsearch

````

# 操作集群

## 查看集群健康

````

get请求

http://172.16.116.121:9200/_cat/health?v

epoch      timestamp cluster status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent 
1516295539 01:12:19  ENT-ES  yellow          1         1     51  51    0    0       51             0                  -                 50.0% 

````
我们可能得到绿色、黄色或红色三种状态。绿色代表一切正常（集群功能齐全）；黄色意味着所有的数据都是可用的，但是某些复制没有被分配（集群功能齐全）； 红色则代表因为某些原因，某些数据不可用,但是可能你需要尽快修复它，因为你有丢失的数据。

## 集群节点

````
get请求

http://172.16.116.121:9200/_cat/nodes?v

host           ip             heap.percent ram.percent load node.role master name     
172.16.116.121 172.16.116.121           36          91 0.81 d         *      TEST_ENT 

````

TEST_ENT代表的是节点的名称

## 列出索引

````
get请求

http://172.16.116.121:9200/_cat/indices?v

health status index                         pri rep docs.count docs.deleted store.size pri.store.size 
yellow open   teacher_platform_unit           5   1       6634           17      2.3mb          2.3mb 


````

## 添加索引

````
get请求

http://172.16.116.121:9200/customer/?pretty

{
  "acknowledged": true
}

get请求

http://172.16.116.121:9200/_cat/indices?v

health status index                         pri rep docs.count docs.deleted store.size pri.store.size 
yellow open   customer                        5   1          0            0       650b           650b 

````

我们现在有一个叫做 customer 的索引，并且它有5个主分片和1份复制（都是默认值），其中包含0个文档。

由于现在我们只有一个节点在运行，那一份复制就分配不了了（为了高可用），直到当另外一个节点加入到这个集群后，才能分配。一旦那份复制在第二个节点上被复制，这个节点(黄色)的健康状态就会变成绿色。


## 索引文档


````

put请求

http://172.16.116.121:9200/customer/external/1?pretty 

参数：
{
  "name": "John Doe"
}

response：

{
  "_index": "customer",
  "_type": "external",
  "_id": "1",
  "_version": 1,
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "created": true
}

````

当你想将文档索引到某个索引的时候，Elasticsearch并不强制要求这个索引被显式地创建。在前面这个例子中，如果customer索引不存在，Elasticsearch将会自动地创建这个索引。


## 查询文档

````

get请求

http://172.16.116.121:9200/customer/external/1?pretty

response:

{
  "_index": "customer",
  "_type": "external",
  "_id": "1",
  "_version": 1,
  "found": true,
  "_source": {
    "name": "John Doe"
  }
}

````

## 删除文档

````
delete请求

http://172.16.116.121:9200/customer/external/1?pretty

response：

{
  "found": true,
  "_index": "customer",
  "_type": "external",
  "_id": "1",
  "_version": 2,
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  }
}

get请求

http://172.16.116.121:9200/customer/external/1?pretty

response：

{
  "_index": "customer",
  "_type": "external",
  "_id": "1",
  "found": false
}

````


## 删除索引

````
delete请求

http://172.16.116.121:9200/customer/?pretty

response:

{
  "acknowledged": true
}


````

## 修改数据

默认情况下，从你索引/更新/删除你的数据动作开始到它出现在你的搜索结果中，大概会有1秒钟的延迟。这和其它的SQL平台不同，它们的数据在一个事务完成之后就会立即可用。


修改前的数据：

````
get请求：
http://172.16.116.121:9200/customer/external/2?pretty  -d

response:

{
  "_index": "customer",
  "_type": "external",
  "_id": "2",
  "_version": 2,
  "found": true,
  "_source": {
    "name": "John Smith"
  }
}

````
使用一个新的document来修改id为2的数据（replace）

````
put请求：
http://172.16.116.121:9200/customer/external/2?pretty  -d

参数：
{
  "name": "John Wall"
}


response：

{
  "_index": "customer",
  "_type": "external",
  "_id": "2",
  "_version": 3,
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "created": false
}

````

id是可选的，如果不指定，则es会产生一个随机的id。由于没有指定id，则使用的是post请求：

````

post请求:

http://172.16.116.121:9200/customer/external/?pretty  -d

参数：

{
  "name": "chenweijie"
}

response：

{
  "_index": "customer",
  "_type": "external",
  "_id": "AWEM3CUozADEKUeU9hxP",
  "_version": 1,
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "created": true
}


````


## 更新文档

Elasticsearch底层并不支持原地更新。在我们想要做一次更新的时候，Elasticsearch先删除旧文档，然后再索引更新的新文档。

````

post请求：

http://172.16.116.121:9200/customer/external/1/_update?pretty

参数：

{
  "doc": { "name": "Jane Ball", "age": 20 }
}

response:

{
  "_index": "customer",
  "_type": "external",
  "_id": "1",
  "_version": 3,
  "_shards": {
    "total": 0,
    "successful": 0,
    "failed": 0
  }
}


````

# 搜索API （复杂查询）

````
REST API 可以通过_search终点(endpoint)来访问

get请求：

http://172.16.116.121:9200/customer/_search?q=*&pretty

response：

{
  "took": 18,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "customer",
        "_type": "external",
        "_id": "AWEM3CUozADEKUeU9hxP",
        "_score": 1,
        "_source": {
          "name": "chenweijie"
        }
      },
      {
        "_index": "customer",
        "_type": "external",
        "_id": "2",
        "_score": 1,
        "_source": {
          "name": "John Wall"
        }
      },
      {
        "_index": "customer",
        "_type": "external",
        "_id": "1",
        "_score": 1,
        "_source": {
          "name": "John tes"
        }
      }
    ]
  }
}

````

以上搜索等价于 请求体方法的等价搜索是：


````
post请求

http://172.16.116.121:9200/customer/_search?pretty

参数：
{
  "query": { "match_all": {} }
}

response：

{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "customer",
        "_type": "external",
        "_id": "AWEM3CUozADEKUeU9hxP",
        "_score": 1,
        "_source": {
          "name": "chenweijie"
        }
      },
      {
        "_index": "customer",
        "_type": "external",
        "_id": "2",
        "_score": 1,
        "_source": {
          "name": "John Wall"
        }
      },
      {
        "_index": "customer",
        "_type": "external",
        "_id": "1",
        "_score": 1,
        "_source": {
          "name": "John tes"
        }
      }
    ]
  }
}

````

我们在customer索引中搜索（ _search终点），并且q=*参数指示Elasticsearch去匹配这个索引中所有的文档。pretty参数仅仅是告诉Elasticsearch返回美观的JSON结果。

- took：Elasticsearch 执行这个搜索的耗时，以毫秒为单位
- timed_out：指明这个搜索是否超时
- _shards：指出多少个分片被搜索了，同时也指出了成功/失败的被搜索的shards 的数量
- hits：搜索结果
- hits.total：匹配查询条件的文档的总数目
- hits.hits：真正的搜索结果数组（默认是前10个文档）


- 搜索中只返回指定的字段：

````
post请求：

http://172.16.116.121:9200/t_tiku_user_record/_search?pretty

参数：

{
  "query": {
    "match_all": {}
  },
  "_source": [
    "knowledgeTreeId",
    "continousDays"
  ]
}


response:

{
  "took": 25,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 11,
    "max_score": 1,
    "hits": [
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10000",
        "_score": 1,
        "_source": {
          "knowledgeTreeId": 16,
          "continousDays": 1
        }
      },
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10006",
        "_score": 1,
        "_source": {
          "knowledgeTreeId": 16,
          "continousDays": 1
        }
      }
      }
    ]
  }
}

````

- 查询条件:stuTotalScore为3000的数据：

````
post请求：

http://172.16.116.121:9200/t_tiku_user_record/_search?pretty


参数：
{
  "query": { "match": { "stuTotalScore": 3000 } }
}


response:

{
  "took": 60,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 1,
    "hits": [
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10000",
        "_score": 1,
        "_source": {
          "knowledgeTreeId": 16,
          "createTime": "2018-01-18T08:12:57.068Z",
          "updateTime": "2018-01-18T08:12:57.068Z",
          "knowledgeNodeId": 12,
          "stuTotalScore": "3000",
          "endTime": "2018-01-18T08:12:57.068Z",
          "tPaperId": 100
        }
      }
    ]
  }
}


````


- 或查询（should）

````
post请求：
http://172.16.116.121:9200/t_tiku_user_record/_search?pretty

参数：
{
  "query": {
    "bool": {
      "should": [
        { "match": { "stuTotalScore": 3000 } },
        { "match": { "stuTotalScore": 3001 } }
      ]
    }
  }
}

response：

{
  "took": 32,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 0.39103588,
    "hits": [
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10001",
        "_score": 0.39103588,
        "_source": {
          "knowledgeTreeId": 16,
          "createTime": "2018-01-18T08:12:57.068Z",
          "updateTime": "2018-01-18T08:12:57.068Z",
          "knowledgeNodeId": 12,
          "stuTotalScore": "3001",
          "endTime": "2018-01-18T08:12:57.068Z",
          "tPaperId": 100
        }
      },
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10000",
        "_score": 0.25427115,
        "_source": {
          "knowledgeTreeId": 16,
          "createTime": "2018-01-18T08:12:57.068Z",
          "updateTime": "2018-01-18T08:12:57.068Z",
          "knowledgeNodeId": 12,
          "stuTotalScore": "3000",
          "endTime": "2018-01-18T08:12:57.068Z",
          "tPaperId": 100
        }
      }
    ]
  }
}


````

- 且查询（must）


````

post请求：
http://172.16.116.121:9200/t_tiku_user_record/_search?pretty

参数：

{
  "query": {
    "bool": {
      "must": [
        { "match": { "knowledgeTreeId": 16 } },
        { "match": { "stuTotalScore": 3001 } }
      ]
    }
  }
}

response：

{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 1.5756677,
    "hits": [
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10001",
        "_score": 1.5756677,
        "_source": {
          "knowledgeTreeId": 16,
          "createTime": "2018-01-18T08:12:57.068Z",
          "updateTime": "2018-01-18T08:12:57.068Z",
          "knowledgeNodeId": 12,
          "stuTotalScore": "3001",
          "endTime": "2018-01-18T08:12:57.068Z",
          "tPaperId": 100
        }
      }
    ]
  }
}

````

## 执行过滤器

文档得分的细节（搜索结果中的_score字段）。这个得分是指定的搜索查询匹配程度的一个相对度量。得分越高，文档越相关，得分越低文档的相关度越低

过滤器的形式提供了另一种查询功能。过滤器在概念上类似于查询，但是它们有非常快的执行速度，这种快的执行速度主要有以下两个原因：

1. 过滤器不会计算相关度的得分，所以它们在计算上更快一些
2. 过滤器可以被缓存到内存中，这使得在重复的搜索查询上，其要比相应的查询快出许多。

````
post请求：

http://172.16.116.121:9200/t_tiku_user_record/_search?pretty

参数：
{
  "query": {
    "filtered": {
      "query": { "match_all": {} },
      "filter": {
        "range": {
          "stuTotalScore": {
            "gte": 3000,
            "lte": 3002
          }
        }
      }
    }
  }
}

response：

{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10000",
        "_score": 1,
        "_source": {
          "knowledgeTreeId": 16,
          "createTime": "2018-01-18T08:12:57.068Z",
          "updateTime": "2018-01-18T08:12:57.068Z",
          "knowledgeNodeId": 12,
          "stuTotalScore": "3000",
          "endTime": "2018-01-18T08:12:57.068Z",
          "tPaperId": 100
        }
      },
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10001",
        "_score": 1,
        "_source": {
          "knowledgeTreeId": 16,
          "createTime": "2018-01-18T08:12:57.068Z",
          "updateTime": "2018-01-18T08:12:57.068Z",
          "knowledgeNodeId": 12,
          "stuTotalScore": "3001",
          "endTime": "2018-01-18T08:12:57.068Z",
          "tPaperId": 100
        }
      },
      {
        "_index": "t_tiku_user_record",
        "_type": "rank",
        "_id": "90-10002",
        "_score": 1,
        "_source": {
          "knowledgeTreeId": 16,
          "createTime": "2018-01-18T08:12:57.068Z",
          "updateTime": "2018-01-18T08:12:57.068Z",
          "knowledgeNodeId": 12,
          "stuTotalScore": "3002",
          "endTime": "2018-01-18T08:12:57.068Z",
          "tPaperId": 100
        }
      }
    ]
  }
}


````

## 执行聚合

聚合查询，主要用于统计相关的，在这里不做详细介绍。具体细节可以翻阅其它资料。


[参考资料](https://endymecy.gitbooks.io/elasticsearch-guide-chinese/content/index.html)



 








