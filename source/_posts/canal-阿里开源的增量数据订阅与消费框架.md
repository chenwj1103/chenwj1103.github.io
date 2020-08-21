---
title: canal-阿里开源的增量数据订阅与消费框架
date: 2017-12-18 13:18:54
tags: mysql,canal
categories: canal

---

  最近拿到一个需求，需要根据数据的变化，实时通知业务方更新redis缓存。所以研究了canal开源框架。
  
  具体业务是，监控库中订单表的某条订单状态发生变化，则将该条数据的id通过kafka的生产者写入topic，业务方从topic中读取订单的id，然后从库里获取订单的数据更新redis。
  
# 准备

 [canal源码](https://github.com/alibaba/canal)中包含canal的文档，server端 client端的 例子 源码包等等。
 

# 来源

 canal是应阿里巴巴存在杭州和美国的双机房部署，存在跨机房同步的业务需求而提出的。目前内部使用的同步，已经支持mysql5.x和oracle部分版本的日志解析

# 支持的业务（数据库同步，增量订阅&消费。）

数据库镜像、数据库实时备份、多级索引（卖家和买家各自分库索引）、**业务cache刷新**、价格变化等重要业务消息、


# 工作原理

## mysql主备复制实现

如下图所示：

 ![mysql主备复制实现](/images/canal/mysql主备复制实现.jpg)
 
从上图看主要分为三个步骤：

1.master将改变记录到二进制日志(binary log)中（这些记录叫做二进制日志事件，binary log events，可以通过show binlog events进行查看）；

 ![查看binlog日志](/images/canal/查看binlog日志.png)

2.slave将master的binary log events拷贝到它的中继日志(relay log)；

3.slave重做中继日志中的事件，将改变反映它自己的数据。

## canal的工作原理

 ![canal的工作原理](/images/canal/canal的工作原理.jpg)
 
原理如下：

1.canal模拟mysql slave的交互协议，伪装自己为mysql slave，向mysql master发送dump协议

2.mysql master收到dump请求，开始推送binary log给slave(也就是canal)

3.canal解析binary log对象(原始为byte流)


# 部署（实例）

## 部署canal-server端

1. 开启mysql的binlog功能，并配置binlog模式为row。在mysql的my.cnf下加入：

````
[mysqld]  
log-bin=mysql-bin #添加这一行就ok  
binlog-format=ROW #选择row模式  
server_id=1 #配置mysql replaction需要定义，不能和canal的slaveId重复  
````

2. canal的原理是模拟自己为mysql的slave，所以这里需要作为mysql slave的权限,而正对已有账户则可以直接grand操作。

````
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
````

3. 下载[canal](https://github.com/alibaba/canal/releases)对应的版需要修改


### 配置

vim canal/conf/example/instance.properties 

````
#################################################
## mysql serverId (要求不与主库的id一样)
canal.instance.mysql.slaveId = 1234

# position info （主库的ip和port）
canal.instance.master.address = 127.0.0.1:3306
canal.instance.master.journal.name = 
canal.instance.master.position = 
canal.instance.master.timestamp = 

#canal.instance.standby.address = 
#canal.instance.standby.journal.name =
#canal.instance.standby.position = 
#canal.instance.standby.timestamp = 

# username/password（主库的账号和密码）
canal.instance.dbUsername = canal
canal.instance.dbPassword = canal
canal.instance.defaultDatabaseName =
canal.instance.connectionCharset = UTF-8

# table regex 
canal.instance.filter.regex = .*\\..*
# table black regex
canal.instance.filter.black.regex =  

#################################################

````

vim canal/conf/canal.properties 


````
#################################################
######### 		common argument		############# 
#################################################
canal.id= 1
canal.ip=
canal.port= 11111
canal.zkServers=
# flush data to zk
canal.zookeeper.flush.period = 1000
# flush meta cursor/parse position to file
canal.file.data.dir = ${canal.conf.dir}
canal.file.flush.period = 1000
## memory store RingBuffer size, should be Math.pow(2,n)
canal.instance.memory.buffer.size = 16384
## memory store RingBuffer used memory unit size , default 1kb
canal.instance.memory.buffer.memunit = 1024 
## meory store gets mode used MEMSIZE or ITEMSIZE
canal.instance.memory.batch.mode = MEMSIZE

## detecing config
canal.instance.detecting.enable = false
#canal.instance.detecting.sql = insert into retl.xdual values(1,now()) on duplicate key update x=now()
canal.instance.detecting.sql = select 1
canal.instance.detecting.interval.time = 3
canal.instance.detecting.retry.threshold = 3
canal.instance.detecting.heartbeatHaEnable = false

# support maximum transaction size, more than the size of the transaction will be cut into multiple transactions delivery
canal.instance.transaction.size =  1024
# mysql fallback connected to new master should fallback times
canal.instance.fallbackIntervalInSeconds = 60

# network config
canal.instance.network.receiveBufferSize = 16384
canal.instance.network.sendBufferSize = 16384
canal.instance.network.soTimeout = 30

# binlog filter config
canal.instance.filter.query.dcl = false
canal.instance.filter.query.dml = false
canal.instance.filter.query.ddl = false
canal.instance.filter.table.error = false
canal.instance.filter.rows = false

# binlog format/image check
canal.instance.binlog.format = ROW,STATEMENT,MIXED 
canal.instance.binlog.image = FULL,MINIMAL,NOBLOB

# binlog ddl isolation
canal.instance.get.ddl.isolation = false

#################################################
######### 		destinations		############# 
#################################################
canal.destinations= example,instance_user_cache_provider
# conf root dir
canal.conf.dir = ../conf
# auto scan instance dir add/remove and start/stop instance
canal.auto.scan = false
canal.auto.scan.interval = 5

canal.instance.global.mode = spring 
canal.instance.global.lazy = false
#canal.instance.global.manager.address = 127.0.0.1:1099
#canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
canal.instance.global.spring.xml = classpath:spring/file-instance.xml
#canal.instance.global.spring.xml = classpath:spring/default-instance.xml


````

 
### 启动

./bin/startup.sh

### 停止

./bin/stop.sh

查看log日志

cat canal/log/canal/canal.log  


## 部署canal-client端

### 创建maven工程，添加pom依赖

````
<dependency>  
    <groupId>com.alibaba.otter</groupId>  
    <artifactId>canal.client</artifactId>  
    <version>1.0.12</version>  
</dependency>  

````

### 示例代码

````
/** 
 * Created by chenwj on 17-11-17. 
 */  
import java.net.InetSocketAddress;  
import java.util.List;  
  
import com.alibaba.otter.canal.client.CanalConnector;  
import com.alibaba.otter.canal.common.utils.AddressUtils;  
import com.alibaba.otter.canal.protocol.Message;  
import com.alibaba.otter.canal.protocol.CanalEntry.Column;  
import com.alibaba.otter.canal.protocol.CanalEntry.Entry;  
import com.alibaba.otter.canal.protocol.CanalEntry.EntryType;  
import com.alibaba.otter.canal.protocol.CanalEntry.EventType;  
import com.alibaba.otter.canal.protocol.CanalEntry.RowChange;  
import com.alibaba.otter.canal.protocol.CanalEntry.RowData;  
import com.alibaba.otter.canal.client.*;  
import org.jetbrains.annotations.NotNull;  
  
public class ClientSample {  
  
    public static void main(String args[]) {  
        // 创建链接   第一个为服务端的ip  第二个为端口 在canal.properties 配置文件中配置 第三个参数为实例的名称   第三个和第四个参数可以不配置
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress(AddressUtils.getHostIp(),  
                11111), "example", "", "");  
        int batchSize = 1000;  
        int emptyCount = 0;  
        try {  
            connector.connect();  
            connector.subscribe(".*\\..*");  
            connector.rollback();  
            int totalEmtryCount = 1200;  
            while (emptyCount < totalEmtryCount) {  
                Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据  
                long batchId = message.getId();  
                int size = message.getEntries().size();  
                if (batchId == -1 || size == 0) {  
                    emptyCount++;  
                    System.out.println("empty count : " + emptyCount);  
                    try {  
                        Thread.sleep(1000);  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                } else {  
                    emptyCount = 0;  
                    // System.out.printf("message[batchId=%s,size=%s] \n", batchId, size);  
                    printEntry(message.getEntries());  
                }  
  
                connector.ack(batchId); // 提交确认  
                // connector.rollback(batchId); // 处理失败, 回滚数据  
            }  
  
            System.out.println("empty too many times, exit");  
        } finally {  
            connector.disconnect();  
        }  
    }  
  
    private static void printEntry(@NotNull List<Entry> entrys) {  
        for (Entry entry : entrys) {  
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {  
                continue;  
            }  
  
            RowChange rowChage = null;  
            try {  
                rowChage = RowChange.parseFrom(entry.getStoreValue());  
            } catch (Exception e) {  
                throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),  
                        e);  
            }  
  
            EventType eventType = rowChage.getEventType();  
            System.out.println(String.format("================> binlog[%s:%s] , name[%s,%s] , eventType : %s",  
                    entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),  
                    entry.getHeader().getSchemaName(), entry.getHeader().getTableName(),  
                    eventType));  
  
            for (RowData rowData : rowChage.getRowDatasList()) {  
                if (eventType == EventType.DELETE) {  
                    printColumn(rowData.getBeforeColumnsList());  
                } else if (eventType == EventType.INSERT) {  
                    printColumn(rowData.getAfterColumnsList());  
                } else {  
                    System.out.println("-------> before");  
                    printColumn(rowData.getBeforeColumnsList());  
                    System.out.println("-------> after");  
                    printColumn(rowData.getAfterColumnsList());  
                }  
            }  
        }  
    }  
  
    private static void printColumn(@NotNull List<Column> columns) {  
        for (Column column : columns) {  
            System.out.println(column.getName() + " : " + column.getValue() + "    update=" + column.getUpdated());  
        }  
    }  
}  


````


### 启动客户端的main方法

启动后看到控制端信息：

````
empty count : 1  
empty count : 2  
empty count : 3  
empty count : 4  

````

触发数据改变

````
create table test (  
uid int (4) primary key not null auto_increment,  
name varchar(10) not null);  
  
insert into test (name) values('10');  

````

日志

````
================> binlog[mysql-bin.000016:3281] , name[canal_test,test] , eventType : INSERT  
uid : 7    update=false  
name : 10    update=false  
empty count : 1  
empty count : 2  

````

# 后记

以上只是简单的单机模式的demo，集群模式和具体的配置文件中的参数详见[配置](https://github.com/alibaba/canal/wiki/AdminGuide)。

其中spring的配置文件采用默认的 canal.instance.global.spring.xml = classpath:spring/file-instance.xml

而xxxx-instance.xml (canal组件的配置定义，可以在多个instance配置中共享) 

如下所示为多个示例

 ![多实例的文件](/images/canal/多实例的配置.jpg)



数据对象格式

````

Entry  
    Header  
        logfileName [binlog文件名]  
        logfileOffset [binlog position]  
        executeTime [binlog里记录变更发生的时间戳,精确到秒]  
        schemaName   
        tableName  
        eventType [insert/update/delete类型]  
    entryType   [事务头BEGIN/事务尾END/数据ROWDATA]  
    storeValue  [byte数据,可展开，对应的类型为RowChange]  
RowChange

isDdl       [是否是ddl变更操作，比如create table/drop table]

sql         [具体的ddl sql]

rowDatas    [具体insert/update/delete的变更数据，可为多条，1个binlog event事件可对应多条变更，比如批处理]

beforeColumns [Column类型的数组，变更前的数据字段]

afterColumns [Column类型的数组，变更后的数据字段]


Column

index

sqlType     [jdbc type]

name        [column name]

isKey       [是否为主键]

updated     [是否发生过变更]

isNull      [值是否为null]

value       [具体的内容，注意为string文本]  

````
 
[更详细讲解](https://github.com/alibaba/canal/wiki)

[参考博客](http://blog.csdn.net/hackerwin7/article/details/37923607)
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
  