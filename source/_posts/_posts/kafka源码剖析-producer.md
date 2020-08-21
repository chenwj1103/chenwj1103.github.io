---
title: kafka源码剖析-producer
copyright: true
date: 2018-04-06 23:53:43
tags: kafka的producer
categories: kafka

---

# kafkaProducer分析

## 发送消息的流程 

![kafka发送消息的整个流程](/images/kafka/producer/kafka发送消息的整个流程.png)

1. producerInterceptors对消息进行拦截
2. Serializer对消息的key和value进行序列化
3. Partitioner为消息选择合适的分区
4. RecordAccumulator收集消息,实现批量发送
5. Sender从RecordAccumulator获取消息
6. 构造ClientRequest
7. 将ClientRequest交给networkClientRequest,准备发送
8. networkClient 将请求放入kafkaChanel缓存
9. 执行网络IO,发送请求
10. 收到响应,调用ClientRequest的回调函数
11. 调用RecordBatch的回调函数,最终调用每个消息上注册的回调函数.


## kafkaProducer接口实现的方法介绍

1. send()方法,发送消息,将消息放入RecordAccumulator暂存,等待发送;
2. flush()方法,刷新操作,等待RecordAccumulator中所有消息发送完成,在刷新完成之前会阻塞调用的线程
3. partitionFor()方法,在kafkaProducer中维护了一个Metadata对象,用于存储kafka集群的元数据,会定时更新,该方法负责从元数据中获取制定topic中的分区信息
4. close()方法,关闭此producer对象,主要操作是设置close标志,等待RecordAccumulator中的消息清空,关闭sender线程.

## kafkaProducer的具体实现:

kafkaProducer的重要参数

````
//clientId生成器
private static final AtomicInteger PRODUCER_CLIENT_ID_SEQUENCE = new AtomicInteger(1);
//clientId
private String clientId;
//分区选择器
private final Partitioner partitioner;
//消息的最大长度,包括消息头.序列化后的key和value
private final int maxRequestSize;
//发送单个消息的缓冲区大小
private final long totalMemorySize;
//kafka的元数据
private final Metadata metadata;
//用于收集缓存消息,等待Sender线程发送
private final RecordAccumulator accumulator;
//发送消息的sender任务,实现了Runnable接口
private final Sender sender;
//执行sender任务发送消息的线程
private final Thread ioThread;
//压缩算法,收集器收集的消息进行压缩
private final CompressionType compressionType;
//key序列化器
private final Serializer<K> keySerializer;
//value序列化器
private final Serializer<V> valueSerializer;
//配置对象
private final ProducerConfig producerConfig;
//等待更新kafka集群元数据的最大时长
private final long maxBlockTimeMs;
//从消息发送到收到ACK相应的最大时长
private final int requestTimeoutMs;
//消息发送之前对消息进行拦截或者修改
private final ProducerInterceptors<K, V> interceptors;

````

![send方法的调用流程](/images/kafka/producer/send方法的调用流程.png)

## ProducerInterceptor

该对象是可以在消息发送之前对其进行拦截或者修改,用户可以实现该接口,然后自定义方法的实现

````
public interface ProducerInterceptor<K, V> extends Configurable {
    public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);
    public void onAcknowledgement(RecordMetadata metadata, Exception exception);
    public void close();
}

````

## kafka集群元数据

### 基本类 

由于kafka生产者在发送消息的时候需要实时的了解kafka分区的相关情况,kafkaProducer中维护了Metadata其中,它用以下三个类封装了集群的相关的元数据

![kafka元数据对象封装的对象](/images/kafka/producer/kafka元数据对象封装的对象.png)

#### kafka集群的元数据

某个topic有几个分区、每个分区的leader副本在哪个节点上、follower副本在哪个节点上、isr集合、这些节点的ip和端口号。

### cluster类

这三个类所组成的对象封装在一个叫做Cluster的类中

````
//kafka集群中节点列表
private final List<Node> nodes;
//
private final Set<String> unauthorizedTopics;
//记录了topicPartition 与partitionInfo之间的关系
private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;
//topic名称与PartitionInfo的映射关系
private final Map<String, List<PartitionInfo>> partitionsByTopic;
private final Map<String, List<PartitionInfo>> availablePartitionsByTopic;
//node与partitionInfo的映射关系
private final Map<Integer, List<PartitionInfo>> partitionsByNode;
//BrokerId与node节点之间的对应关系
private final Map<Integer, Node> nodesById;

````
### metadata类

````
//两次发出更新clsuter保存的元数据信息的最小时间差
private final long refreshBackoffMs;
//每隔多久更新一次
private final long metadataExpireMs;
//kafka集群元数据的版本号,每更新一次,值加1
private int version;
//上一次更新元数据的时间戳
private long lastRefreshMs;
//上一次更新元数据成功的时间戳
private long lastSuccessfulRefreshMs;
//记录kafka集群的元数据
private Cluster cluster;
private boolean needUpdate;
//topic最新的元数据
private final Set<String> topics;
private final List<Listener> listeners;
private boolean needMetadataForAllTopics;

````
metadata类中主要的waitOnMetadata()方法主要时触发元数据的更新

````
  private long waitOnMetadata(String topic, long maxWaitMs) throws InterruptedException {
        // add topic to metadata topic list if it is not there already.
        if (!this.metadata.containsTopic(topic))
            this.metadata.add(topic);
            
        //成功获取分区的详细信息
        if (metadata.fetch().partitionsForTopic(topic) != null)
            return 0;

        long begin = time.milliseconds();
        long remainingWaitMs = maxWaitMs;
        while (metadata.fetch().partitionsForTopic(topic) == null) {
            //设置needupdate,获取当前元数据版本号
            int version = metadata.requestUpdate();
            sender.wakeup();//唤醒sender线程
            //阻塞等待元数据更新完毕
            metadata.awaitUpdate(version, remainingWaitMs);
            long elapsed = time.milliseconds() - begin;
            if (elapsed >= maxWaitMs)
                throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
            if (metadata.fetch().unauthorizedTopics().contains(topic))
                throw new TopicAuthorizationException(topic);
            remainingWaitMs = maxWaitMs - elapsed;
        }
        return time.milliseconds() - begin;
    }


````


## Serializer和Deserializer

客户端发送的消息的key和value都是byte数组,这两个接口实现了将java对象序列化和反序列化为byte数组的功能.

## partitioner

kafkaProducer.send()方法的下一步操作是选择消息的分区.DefaultPartitioner中对partition的实现.直接上代码


````
    //初始化一个随机数,是线程安全的AtomicInteger对象
    private final AtomicInteger counter = new AtomicInteger(new Random().nextInt());
    
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        //从cluster中获取分片信息
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (keyBytes == null) { //对于没有key的情况,递增counter
            int nextValue = counter.getAndIncrement();
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = DefaultPartitioner.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                return DefaultPartitioner.toPositive(nextValue) % numPartitions;
            }
        } else {
            // hash the keyBytes to choose a partition
            //对于有key的情况,对key进行hash(murmur2的hash算法),然后与分区数量取模
            return DefaultPartitioner.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }

````

# RecordAccumulator分析

kafkaProducer可以有同步和异步两种方式发送消息,两者的底层实现都是异步的.主线程send()方法发送消息的时候,现将消息放到RecordAccumulator中缓存,然后主线程可以从send()方法中返回了,其实消息没有真正的发送,而是缓存在RecordAccumulator对象中,业务线程不断的通过send方法追加消息,达到一定条件会唤醒Sender线程,发送RecordAccumulator中的消息.

RecordAccumulator至少有一个业务线程和一个Sender线程并发操作,所以RecordAccumulator时线程安全的.

RecordAccumulator中有一个以TopicPartition为key的ConcurrentMap,每个value都是Deque<RecordBatch>,每个RecordBatch都拥有一个MemoryRecords的对象的引用,MemoryRecords才是消息最终存放的地方

````

public final class RecordAccumulator{

    private final ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches;

}

````



## MemoryRecords对象

该对象很重有四个字段比较重要
````
//压缩器,对消息数据进行压缩,将压缩后的数据输出到buffer
private final Compressor compressor;
//记录buffer字段最多可以写入多少字节的数据
private final int writeLimit;
//用于保存消息数据的javaNIO ByteBuffer
private ByteBuffer buffer;
//MemoryRecords对象是只读模式,还是可写模式,该对象发送前时,将其设置为只读模式.
private boolean writable;

````

compressor中重要的字段有bufferStream和appendStream.

````
// 压缩操作
public Compressor(ByteBuffer buffer, CompressionType type) {
        this.type = type;

        if (type != CompressionType.NONE) {
            // for compressed records, leave space for the header and the shallow message metadata
            // and move the starting position to the value payload offset
            buffer.position(initPos + Records.LOG_OVERHEAD + Record.RECORD_OVERHEAD);
        }

        // create the stream
        bufferStream = new ByteBufferOutputStream(buffer);
        //根据压缩类型创建合适的压缩流
        appendStream = wrapForOutput(bufferStream, type, COMPRESSION_DEFAULT_BUFFER_SIZE);
    }
    
//包装压缩流
    public static DataOutputStream wrapForOutput(ByteBufferOutputStream buffer, CompressionType type, int bufferSize) {
        try {
            switch (type) {
                case NONE:
                    return new DataOutputStream(buffer);
                case GZIP:
                    return new DataOutputStream(new GZIPOutputStream(buffer, bufferSize));
                case SNAPPY:
                    try {
                        OutputStream stream = (OutputStream) snappyOutputStreamSupplier.get().newInstance(buffer, bufferSize);
                        return new DataOutputStream(stream);
                    } catch (Exception e) {
                        throw new KafkaException(e);
                    }
                case LZ4:
                    try {
                        OutputStream stream = (OutputStream) lz4OutputStreamSupplier.get().newInstance(buffer);
                        return new DataOutputStream(stream);
                    } catch (Exception e) {
                        throw new KafkaException(e);
                    }
                default:
                    throw new IllegalArgumentException("Unknown compression type: " + type);
            }
        } catch (IOException e) {
            throw new KafkaException(e);
        }
    }


 compressor提供了一系列put*()方法,向appendStream中写入数据,这是个装饰器模式,通过bufferStream装饰,添加自动扩容的功能.
 

````


通过emptyRecords()方法得到MemoryRecords对象,

append() 判断MemoryRecords对象是否为可写模式,然后调用Compressor.put*()方法,将消息写入到ByteBuffer对象

hashRoomFor() 估计对象中是否有空间继续吸入数据

close()方法,将buffer字段指向另一个ByteBuffer对象,将writable设置为false.

sizeInBytes()方法:返回MemoryRecords.buffer()的大小



## RecordBatch

````
public final class RecordBatch {

    // 记录的个数
    public int recordCount = 0;
    //最大record的字节数
    public int maxRecordSize = 0;
    //尝试发送当前recordBatch的次数
    public volatile int attempts = 0;
    //最后一次尝试发送的时间戳    
    public long lastAttemptMs;
    //用来存储数据的MemoryRecords对象
    public final MemoryRecords records;
    //MemoryRecords对象中存储的数据会批量发送给topicPartition
    public final TopicPartition topicPartition;
    //标示RecordBatch状态的future对象
    public final ProduceRequestResult produceFuture;
    //上一次追加消息的时间
    public long lastAppendTime;
    //thunks对象的集合
    private final List<Thunk> thunks;
    //某消息在recordBatch对象中的偏移量
    private long offsetCounter = 0L;
    //是否正在进行重试
    private boolean retry;
    }

````

RecordBatch主要用于封装MemoryRecords以及其他的一些统计类型的信息。

ProduceRequestResult: 完成生产者请求的一个结果类，但是该类并没有根据并发库下的Future来实现而是根据CountDownLatch来实现。当RecordBatch中全部消息被正常响应，或超市或关闭生产者时，会调用done方法标记完成，可以通过error字段区分是异常完成还是正常完成

### bufferPool

ByteBuffer的创建和释放是比较消耗资源的,为了实现资源的高效利用,基本上每个成熟的框架或者工具都有一套管理机制.kafka使用bufferPool进行管理.

bufferPool对象只针对特定大小的byteBuffer字段进行管理,memoryRecords的大小由RecordAccumulator.bathSize字段指定


````

public final class BufferPool {
    
    //整个bufferPool的大小
    private final long totalMemory;
    private final int poolableSize;
    //因为有多线程并发分配和回收byteBuffer,所以使用所控制并发线程的安全
    private final ReentrantLock lock;
    //是一个队列,缓存了指定大小的byteBuffer对象
    private final Deque<ByteBuffer> free;
    //记录因申请不到足够空间而阻塞的线程,此队列中实际记录的是阻塞线程对应的Condition对象
    private final Deque<Condition> waiters;
    //整个可用空间的大小,totalMemory-free
    private long availableMemory;
    private final Metrics metrics;
    private final Time time;
    private final Sensor waitTime;
    
    }


````


allocate方法负责从bufferPool中申请ByteBuffer,当缓冲池中空间不足时,就会阻塞调用线程.

````
//申请空间

public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
        if (size > this.totalMemory)
            throw new IllegalArgumentException("Attempt to allocate " + size
                                               + " bytes, but there is a hard limit of "
                                               + this.totalMemory
                                               + " on memory allocations.");
    //加锁同步
        this.lock.lock();
        try {
            // check if we have a free buffer of the right size pooled
            if (size == poolableSize && !this.free.isEmpty())
                return this.free.pollFirst();

            // now check if the request is immediately satisfiable with the
            // memory on hand or if we need to block
            int freeListSize = this.free.size() * this.poolableSize;
            if (this.availableMemory + freeListSize >= size) {
                // we have enough unallocated or pooled memory to immediately
                // satisfy the request
                freeUp(size);
                this.availableMemory -= size;
                lock.unlock();
                return ByteBuffer.allocate(size);
            } else {
                // we are out of memory and will have to block
                int accumulated = 0;
                ByteBuffer buffer = null;
                Condition moreMemory = this.lock.newCondition();
                long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                this.waiters.addLast(moreMemory);
                // loop over and over until we have a buffer or have reserved
                // enough memory to allocate one
                while (accumulated < size) {
                    long startWaitNs = time.nanoseconds();
                    long timeNs;
                    boolean waitingTimeElapsed;
                    try {
                        waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                    } catch (InterruptedException e) {
                        this.waiters.remove(moreMemory);
                        throw e;
                    } finally {
                        long endWaitNs = time.nanoseconds();
                        timeNs = Math.max(0L, endWaitNs - startWaitNs);
                        this.waitTime.record(timeNs, time.milliseconds());
                    }

                    if (waitingTimeElapsed) {
                        this.waiters.remove(moreMemory);
                        throw new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");
                    }

                    remainingTimeToBlockNs -= timeNs;
                    // check if we can satisfy this request from the free list,
                    // otherwise allocate memory
                    if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                        // just grab a buffer from the free list
                        buffer = this.free.pollFirst();
                        accumulated = size;
                    } else {
                        // we'll need to allocate memory, but we may only get
                        // part of what we need on this iteration
                        freeUp(size - accumulated);
                        int got = (int) Math.min(size - accumulated, this.availableMemory);
                        this.availableMemory -= got;
                        accumulated += got;
                    }
                }

                // remove the condition for this thread to let the next thread
                // in line start getting memory
                Condition removed = this.waiters.removeFirst();
                if (removed != moreMemory)
                    throw new IllegalStateException("Wrong condition: this shouldn't happen.");

                // signal any additional waiters if there is more memory left
                // over for them
                if (this.availableMemory > 0 || !this.free.isEmpty()) {
                    if (!this.waiters.isEmpty())
                        this.waiters.peekFirst().signal();
                }

                // unlock and return the buffer
                lock.unlock();
                if (buffer == null)
                    return ByteBuffer.allocate(size);
                else
                    return buffer;
            }
        } finally {
            if (lock.isHeldByCurrentThread())
                lock.unlock();
        }
    }



// 释放空间

public void deallocate(ByteBuffer buffer, int size) {
    lock.lock();
    try {
        if (size == this.poolableSize && size == buffer.capacity()) {
            buffer.clear();
            this.free.add(buffer);
        } else {
            this.availableMemory += size;
        }
        Condition moreMem = this.waiters.peekFirst();
        if (moreMem != null)
            moreMem.signal();
    } finally {
        lock.unlock();
    }
}

````


### RecordAccumulator详解

````
public final class RecordAccumulator {

    private volatile boolean closed;
    private final AtomicInteger flushesInProgress;
    private final AtomicInteger appendsInProgress;
    //每个recordBath底层ByteBuffer的大小
    private final int batchSize;
    //压缩类型
    private final CompressionType compression;
    private final long lingerMs;
    private final long retryBackoffMs;
    //bufferPool对象
    private final BufferPool free;
    private final Time time;
    //topicPartition与recordBath集合的映射关系,CopyOnWriteMap时线程安全的集合
    private final ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches;
    //未发送完成的RecordBatch集合
    private final IncompleteRecordBatches incomplete;
    // The following variables are only accessed by the sender thread, so we don't need to protect them.
    private final Set<TopicPartition> muted;
    //使用drain方法批量导入RecordBath时,为了防止饥饿,记录上次发送停止时的位置,下次饥饿从此位置开始
    private int drainIndex;
    
    Map<Integer, List<RecordBatch>> drain(Cluster cluster,
                                                     Set<Node> nodes,
                                                     int maxSize,
                                                     long now)
                                                     
    public ReadyCheckResult ready(Cluster cluster, long nowMs)                                                      
                                                     
    private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Callback callback, Deque<RecordBatch> deque)
    
    public RecordAppendResult append(TopicPartition tp,
                                               long timestamp,
                                               byte[] key,
                                               byte[] value,
                                               Callback callback,
                                               long maxTimeToBlock) throws InterruptedException                                  
    

}


````

kafkaProducer.send()方法最终会调用recordsAccumulator.append()方法将消息追加到RecordAccumulator中

1. 首先在batches集合中查找topicPartition对应的Deque,查找不到则创建新的Deque,并添加到batches集合中;
2. 对Deque加锁(使用synchronize关键字加锁)
3. 使用tryAppend()方法,尝试向Deque中最后一个RecordBatch追加record
4. synchronize块结束,自动解锁
5. 追加成功,返回RecordAppendResult,
6. 追加失败,则尝试从bufferPool中申请新的byteBuffer
7. 对Deque加锁,
8. 追加成功,则返回,追加失败则使用第五步得到的ByteBuffer创建RecordBatch,
9. 将Record追加到新建的RecordBatch,并将新建的RecordBatch追加到Deque尾部
10. 将新建的RecordBatch追加到incomplete集合中
11. synchronize代码块结束,自动解锁
12. 返回RecordAppendResult,RecordAppendResult中的字段作为唤醒Sender线程的条件.

````

    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        try {
            // check if we have an in-progress batch
            Deque<RecordBatch> dq = getOrCreateDeque(tp);
            synchronized (dq) {
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null)
                    return appendResult;
            }

            // we don't have an in-progress record batch try to allocate a new batch
            int size = Math.max(this.batchSize, Records.LOG_OVERHEAD + Record.recordSize(key, value));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
            synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");

                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null) {
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                    free.deallocate(buffer);
                    return appendResult;
                }
                MemoryRecords records = MemoryRecords.emptyRecords(buffer, compression, this.batchSize);
                RecordBatch batch = new RecordBatch(tp, records, time.milliseconds());
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, callback, time.milliseconds()));

                dq.addLast(batch);
                incomplete.add(batch);
                return new RecordAppendResult(future, dq.size() > 1 || batch.records.isFull(), true);
            }
        } finally {
            appendsInProgress.decrementAndGet();
        }
    }


````


drain方法是将各个node节点的RecordBatch进行分组.在网络IO层面发送的时候,生产者时面向node层面发送消息数据.

````
public Map<Integer, List<RecordBatch>> drain(Cluster cluster,
                                                 Set<Node> nodes,
                                                 int maxSize,
                                                 long now) {
        if (nodes.isEmpty())
            return Collections.emptyMap();
        //转换后的结果
        Map<Integer, List<RecordBatch>> batches = new HashMap<>();
        for (Node node : nodes) { //遍历指定ready node集合
            int size = 0;
            //获取当前node上的分区集合
            List<PartitionInfo> parts = cluster.partitionsForNode(node.id());
            //记录要发送的RecordBatch drainIndex 时Batches的下标,记录上次发送停止的位置, 如果一直从0开发发送,会造成其它的分区饥饿
            List<RecordBatch> ready = new ArrayList<>();
            /* to make starvation less likely this loop doesn't start at 0 */
            int start = drainIndex = drainIndex % parts.size();
            do {
            //获取分区的详细情况
                PartitionInfo part = parts.get(drainIndex);
                TopicPartition tp = new TopicPartition(part.topic(), part.partition());
                // Only proceed if the partition has no in-flight batches.
                if (!muted.contains(tp)) {
                    Deque<RecordBatch> deque = getDeque(new TopicPartition(part.topic(), part.partition()));
                    if (deque != null) {
                        synchronized (deque) {
                            RecordBatch first = deque.peekFirst();
                            if (first != null) {
                                boolean backoff = first.attempts > 0 && first.lastAttemptMs + retryBackoffMs > now;
                                // Only drain the batch if it is not during backoff period.
                                if (!backoff) {
                                    if (size + first.records.sizeInBytes() > maxSize && !ready.isEmpty()) {
                                        // there is a rare case that a single batch size is larger than the request size due
                                        // to compression; in this case we will still eventually send this batch in a single
                                        // request
                                        break;
                                    } else {
                                        RecordBatch batch = deque.pollFirst();
                                        batch.records.close();
                                        size += batch.records.sizeInBytes();
                                        ready.add(batch);
                                        batch.drainedMs = now;
                                    }
                                }
                            }
                        }
                    }
                }
                this.drainIndex = (this.drainIndex + 1) % parts.size();
            } while (start != drainIndex);
            batches.put(node.id(), ready);
        }
        return batches;
    }



````



# Sender分析

流程: 首先根据RecordAccumulator的缓存情况,筛选出可以向哪些node节点发送信息,即之前介绍的ready方法,然后根据生产者与各个节点的连接情况,过滤node节点,生成相应的请求,每个node节点只生成一个请求,最后调用netWorkClient将将请求发送出去.

Sender实现了Runnable接口,并运行在单独的ioThread中,sender的run()方法调用了其重载run(long),这是sender方法的核心.

![sender方法的时序图](/images/kafka/producer/sender方法的时序图.png)

1. 从metadata获取kafka集群的元数据
2. 调用RecordAccumulator.read()方法,根据RecordAccumulator的缓存情况,选出可以想那些node节点发送消息,返回readyCheckResult对象
3. 如果readyCheckResult中标识有unknownLeadersExist,则调用Metadata的requestUpdate方法,标记需要更新kafka的集群消息.
4. 针对readyCheckResult中readynodes集合,循环调用netWorkClient.Ready()方法,目的时检查网络IO是否符合发送消息的条件,不符合的将会从readyNodes节点中删除
5. 调用RecordAccumulator.drain()方法获取待发送的消息集合.
6. 调用RecordAccumulator.abortExpiredBatches()方法处理超时的消息,具体是遍历全部的recordBatch 调用maybeExpire()进行处理,如果已经超时调用recordBatch.done()方法出发自定义的callback,将RecordBatch从队列中移除,释放ByteBuffer.
7. 调用Sender.createProduceRequests()方法将待发送的消息封装成ClientRequest
8. 调用networkClient.send()方法,将ClientRequest写入kafkaChanel的send字段.
9. 调用netWorkClient.poll()方法,将kafkaChannel.send字段中保存的clientRequest发送出去,还会处理客户端发回的相应,处理超时的请求,调用用户自定义的callback

## 创建请求

![produce request和produce response](/images/kafka/producer/produce request和produce response.png)

![协议各个字段的解释](/images/kafka/producer/协议各个字段的解释.png)


createProduceRequests的代码:

````
    /**
     * Transfer the record batches into a list of produce requests on a per-node basis
     */
    private List<ClientRequest> createProduceRequests(Map<Integer, List<RecordBatch>> collated, long now) {
        List<ClientRequest> requests = new ArrayList<ClientRequest>(collated.size());
        for (Map.Entry<Integer, List<RecordBatch>> entry : collated.entrySet())
        //调用produceRequest方法,将发往同一node的RecordBatch分装成一个ClientRequest对象
            requests.add(produceRequest(now, entry.getKey(), acks, requestTimeout, entry.getValue()));
        return requests;
    }

    /**
     * Create a produce request from the given record batches
     */
    private ClientRequest produceRequest(long now, int destination, short acks, int timeout, List<RecordBatch> batches) {
        Map<TopicPartition, ByteBuffer> produceRecordsByPartition = new HashMap<TopicPartition, ByteBuffer>(batches.size());
        final Map<TopicPartition, RecordBatch> recordsByPartition = new HashMap<TopicPartition, RecordBatch>(batches.size());
        //将recordBatch 列表按照partition分类,整理成上述两个集合
        for (RecordBatch batch : batches) {
            TopicPartition tp = batch.topicPartition;
            produceRecordsByPartition.put(tp, batch.records.buffer());
            recordsByPartition.put(tp, batch);
        }
        //创建produceRequest 和requestSend
        ProduceRequest request = new ProduceRequest(acks, timeout, produceRecordsByPartition);
        RequestSend send = new RequestSend(Integer.toString(destination),
                                           this.client.nextRequestHeader(ApiKeys.PRODUCE),
                                           request.toStruct());
        RequestCompletionHandler callback = new RequestCompletionHandler() {
            public void onComplete(ClientResponse response) {
                handleProduceResponse(response, recordsByPartition, time.milliseconds());
            }
        };

        return new ClientRequest(now, acks != 0, send, callback);
    }

````


## KSelector

在介绍netWorkClient之前,先了解其结构,它是属于kafka自己的包下的结构.


![kselector](/images/kafka/producer/kselector.png)


## networkClient
![networkClient](/images/kafka/producer/networkClient.png)











