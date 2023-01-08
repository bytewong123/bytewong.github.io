---
title: kafka知识点汇总
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["消息队列"]
tags: ["消息队列","kafka"]
---

# kafka知识点汇总
#技术/消息队列
## topic
每条发布到kafka集群的消息都有一个类别，这个类别被称为topic
- 如果某topic有N个partition，集群有N个broker，那么每个broker存储该topic的一个partition
- 如果某topic有N个partition，集群有(N+M)个broker，那么其中有N个broker存储该topic的一个partition，剩下的M个broker不存储该topic的partition数据
- 如果某topic有N个partition，集群中broker数目少于N个，那么一个broker存储该topic的一个或多个partition

## partition
### leader
任何partition只有一个leader，只有leader是对外提供读写服务的
### follower
Leader副本接收到数据之后，Follower副本会不停的给他发送请求尝试去拉取最新的数据，拉取到自己本地后，写入磁盘中
### ISR
In-Sync Replicas，每个partition都有一个ISR
保持同步的副本，是一个列表，记录了跟leader保持同步的follower
- 假设replica.lag.max.messages设置为4，表明只要follower落后leader不超过3，就不会从isr中移除
- replica.lag.time.max设置为500 ms，表明只要follower向leader发送请求时间间隔不超过500 ms，就不会被标记为死亡,也不会从同步isr中移除
### 部署方式
- 每个partition都有多个副本，相同partition的各个副本分布在不同的broker上
- partition的多副本分为leader和follower；
- 有一个leader和多个follower
### 消费
- partition只能同一个group中的同一个consumer消费，如果想要重复消费，那么需要其他的组来消费
#### 什么情况下会重复消费？
消息处理后，offset未提交
## consumer
### rebalance
Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 consumer 如何达成一致，来分配订阅 Topic 的每个分区
#### 触发条件：
1. consumer加入或离开
2. coordinator挂了，集群选举出新的coordinator
3. topic的partition新加
4. consumer调用unsubscrible()，取消topic的订阅
#### 影响：
Consumer被踢出消费组，可能还没有提交offset，Rebalance时会Partition重新分配其它Consumer,会造成重复消费
#### 避免rebalance的方式：
1. 未能及时发送心跳造成的rebalance
- session.timeout.ms 当broker多久没有收到consumer的心跳请求后就触发rebalance，默认值是10s（服务端逻辑）
- heartbeat.interval.ms  控制consumer发送心跳请求频率的参数，每次发送心跳的间隔（客户端逻辑）
> 每个consumer 都会根据 heartbeat.interval.ms 参数指定的时间周期性地向group coordinator发送 hearbeat；
> 由于心跳请求是在poll()拉取消息的方法中执行的，因此如果当前批次处理消息耗时太长，就会导致consumer没有机会按时发送心跳，broker认为消费者已死，触发rebalance。在0.10.1.0或更新的版本中解决了这个问题，心跳请求会在单独的线程中发送，因此就不会出现因为消息处理过长而发不出心跳的问题了
2. consumer消费超时而造成的rebalance
- max.poll.interval.ms  这个参数定义了两次poll()之间的最大间隔，默认值为5分钟。如果超过这个间隔同样会触发rebalance。能否在5min内处理完还取决于每次拉取了多少条消息
- max.poll.records 这个参数定义了poll()方法最多可以返回多少条消息，默认值为500。

### group coordinator
#### consumer group partition分配过程（rebalance过程）
1. 每个consumer group有一个 group coordinator。对于每个consumer group，Kafka集群为其从broker集群中选择一个broker作为其coordinator
2. consumer group里面的各个consumer都向 group coordinator发送JoinGroup请求，这样group coordinator就有了所有consumer的成员信息
3. group coordinator从中选出一个consumer作为Leader consumer，并告诉Leader consumer说：你拿着这些成员信息和我给你的topic分区信息去安排一下哪些consumer负责消费哪些分区吧
4. Leader consumer根据配置的分配策略(由参数partition.assignment.strategy指定)为各个consumer计算好了各自待消费的分区。partition的分配策略和分配结果其实是由client决定的，而不是由coordinator决定的
> 让client分配不让server分配的原因：如果让server分配，一旦需要新的分配策略，server集群要重新部署，这对于已经在线上运行的集群来说，代价是很大的；而让client分配，server集群就不需要重新部署了
5. leader通过SyncGroup消息，把分配结果发给coordinator，其他consumer也发送SyncGroup消息，获得这个分配结果

#### rebalance generation
它表示了rebalance之后的一届成员，主要是用于保护consumer group，隔离无效offset提交的。比如上一届的consumer成员是无法提交位移到新一届的consumer group中。每次group进行rebalance之后，generation号都会加1，表示group进入到了一个新的版本

#### offset管理
1. kafka把所有topic的所有partition提交的offset保存到了一个__consumer_offset的topic下，包含两个字段：
> key: Consumer Group, topic, partition
> Payload:Offset, metadata, timestamp

2. 这个__consumer_offset 有50个分区，通过将group的id哈希值%50的值来确定要保存到那一个分区
3. offset提交消息会根据消费组的key(消费组名称)进行分区. 对于一个给定的消费组,它的所有消息都会发送到唯一的broker(即Coordinator)
4. 该broker还会维护一张offset表，便于快速查询offset

Coordinator上负责管理offset的组件是Offset manager。负责存储,抓取,和维护消费者的offsets. 每个broker都有一个offset manager实例. 有两种具体的实现:
1. ZookeeperOffsetManager: 调用zookeeper来存储和接收offset（老版本的位移管理）。
2. DefaultOffsetManager: 提供消费者offsets内置的offset管理。
通过在config/server.properties中的offset.storage参数选择。

**DefaultOffsetManager**
除了将offset作为logs保存到磁盘上,DefaultOffsetManager维护了一张能快速服务于offset抓取请求的consumer offsets表。

消费开始时，拉取offset的过程
1. 发送请求至任意一个broker；
2. broker确认自己是否为该group的offset存储的partition leader，如果不是就把请求转至对应的partition；
3. leader partition处理后，返回给转发过来的broker，broker再把结果返回给consumer

## kafka性能
1. 顺序读写
- kafka将来自Producer的数据，顺序追加在partition，partition就是一个文件，以此实现顺序写入
- Consumer从broker读取数据时，因为自带了偏移量，接着上次读取的位置继续读，以此实现顺序读
2. 零拷贝
传统模式下，当需要对一个文件进行传输的时候，其具体流程细节如下：
* 调用 read 函数，文件数据被 copy 到内核缓冲区
* read 函数返回，文件数据从内核缓冲区 copy 到用户缓冲区
* write 函数调用，将文件数据从用户缓冲区 copy 到内核与 socket 相关的缓冲区
* 数据从 socket 缓冲区 copy 到相关协议引擎。

sendfile 系统调用则提供了一种减少以上多次 copy：
* sendfile 系统调用，文件数据被 copy 至内核缓冲区
* 再从内核缓冲区 copy 至内核中 socket 相关的缓冲区

3. mmap
对磁盘文件发起IO操作，首先从磁盘上把数据读取到内核IO缓冲区里去，然后再从内核IO缓存区里读取到用户进程私有空间里去，然后我们才能拿到这个文件里的数据。为了读取磁盘文件里的数据，是不是发生了两次数据拷贝。写文件同理。
mmap方法为我们提供了将文件的部分或全部映射到内存地址空间的能力，同当这块内存区域被写入数据之后[dirty]，操作系统会在一定时刻把这些数据写入到文件中。
刚开始建立映射的时候，并没有任何的数据拷贝操作，其实磁盘文件还是停留在那里，只不过他把物理上的磁盘文件的一些地址和用户进程私有空间的一些虚拟内存地址进行了一个映射
这个mmap技术在进行文件映射的时候，一般有大小限制，在1.5GB~2GB之间，所以在很多消息中间件，会限制文件的大小。
**PageCache，实际上在这里就是对应于虚拟内存**
接下来就可以对这个已经映射到内存里的磁盘文件进行读写操作了，比如要写入消息到文件，你先把一文件通过MappedByteBuffer的map()函数映射其地址到你的虚拟内存地址。
接着就可以对这个MappedByteBuffer执行写入操作了，写入的时候他会直接进入PageCache中，然后过一段时间之后，由os的线程异步刷入磁盘中。
使用了页缓存，只有一次数据拷贝的过程，就是从PageCache里拷贝到磁盘文件。

## 文件存储
### 存储方式
按partition存储，同一个topic下有多个不同partition，每一个partition为一个文件夹，partiton目录命名规则为topic名称+partition序号。每一个partion(文件夹)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件里

segment文件，分为index file和data file
- index file
索引文件存储offset与对应的message在data文件中的物理偏移
- data file
partion全局的第一个segment data文件名从0开始，是00000.log，19位数字字符长度，没有数字用0填充
每一个segment文件名称为上一个segment文件最后一条消息的offset值

如何通过offset定位message
1. 定位segment index文件
2. 通过index文件中的offset查找data文件中的message

### kafka高效文件存储的特点
1. Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
2. 通过索引信息可以快速定位message和确定response的最大大小。
3. 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
4. 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。

## kafka为什么要使用主写主读的方式
1. 数据一致性问题：数据同步需要时间，应用读取从节点中的数据的值并不为最新的值，由此便产生了数据不一致的问题
2.  Kafka 中，主从同步会比 Redis 更加耗时，它需要经历网络→主节点内存→主节点磁盘→网络→从节 点内存→从节点磁盘这几个阶段，由于内存和磁盘这个环节，对延时敏感的应用而言，主写从读的功能并不太适用

# 常见问题
## kafka如何保证不丢失消息
发送端：
1. 复制因子：创建topic的时候指定复制因子大于1时，一个分区被分配到一个broker上，同时会在其他broker上维护一个分区副本；
2. isr列表：分区及其副本分别为leader和follower，leader对外提供读写服务，follower会向leader发送同步请求，拉取最新的数据，如果follower和leader的消息差距保持在一定范围之内，那么这个follower在isr列表内；当分区leader所在broker宕机，会从isr列表中选举一个follower作为新的leader提供服务
3. 通过kafka的acks参数可以控制消息的发送行为，acks可选的值有0、1、all；当设置为0时，生产者消息发送成功即为成功，不关心是否写入到磁盘及后续操作；当设置为1时，消息发送到分区leader后写入磁盘即为成功；当设置为all时，消息发送到分区leader并写入磁盘后，同步给isr列表中的所有分区副本后即为成功
消费端：
1. 如果在消息处理完成前就提交了offset，那么就有可能造成数据的丢失。enable.auto.commit=false 关闭自动提交位移 在消息被完整处理之后再手动提交位移

## zk的作用
zookeeper 是一个分布式的协调组件，早期版本的kafka用zk做meta信息存储，consumer的消费状态，group的管理以及 offset的值。考虑到zk本身的一些因素以及整个架构较大概率存在单点问题，新版本中逐渐弱化了zookeeper的作用。新的consumer使用了kafka内部的group coordination协议，也减少了对zookeeper的依赖，

但是broker依然依赖于ZK，zookeeper 在kafka中还用来选举controller 和 检测broker是否存活等等。

## Kafka 消息数据积压，消费能力不足怎么处理？
1. 可以考虑增加Topic的分区数，并且同时提升消费组的消费者数量，消费者数=分区数。（两者缺一不可）
2. 如果是下游的数据处理不及时：提高每批次拉取的数量。批次拉取数据过少（拉取数据/处理时间<生产速度），使处理的数据小于生产的数据，也会造成数据积压。

### kafka重复消费如何处理？
正常情况下在consumer真正消费完消息后应该发送ack，通知broker该消息已正常消费，从queue中剔除
当ack因为网络原因无法发送到broker，broker会认为词条消息没有被消费，此后会开启消息重投机制把消息再次投递到consumer。
使用外部介质：
1. 数据库表：处理消息前，使用消息主键在表中带有约束的字段中insert
2. Redis：使用分布式锁。



