---
layout: post
title: "重识Kafka-02:架构设计"
permalink: /kafka_02/design/
---

# 指南
上一篇: [重识Kafka_01:背景](https://spikeryang.github.io/kafka_01/background/)  
本篇的内容对应论文的3.架构设计

# 架构 & 设计
## Kafka基础概念
topic: 某一个类型的消息流, producer发送消息到一个topic  
broker: 已发送的消息存储在brokers  
consumer可以订阅topic，从brokers pull消息  
partition: 一个topic被分成很多partition, 一个borker存储一个或多个partition  

## 单partition存储的效率
- 简单存储: 一个partition逻辑上对应一个无限增长的append-only文件。这个抽象的文件实际上由很多固定大小的文件组成（例如1GB）。  
- Flush: 落盘。每收到一条消息，broker会把新消息添加到文件末尾，积累一定数量的消息或者经过一定时间才会把改动写入磁盘，这样减少了磁盘io。落盘后，这些新消息才会对consumer可见。  
- id: kafka消息没有传统意义上的 **id** ，而是把消息在log里的逻辑offset作为id，这样的设计避免了维护消息id和消息在log中具体位置的对应关系。消息id是递增而不连续的。通过当前消息的id加上消息长度，则获得了下一条消息的id。   
- 顺序消费: consumer从一个partition顺序消费，如果consumer提交了一个offset，意味着这个offset之前的消息都已经消费。(这个原则在Raft的设计中也有使用)  
- 避免内存缓存: Kafka没有做应用层的缓存，而是依赖底层文件系统的页缓存。这样设计的优势在于避免双重缓存，而且当broker进程重启的时候可以重新获取到warm cache（我理解这里指的是如果使用内存缓存，机器重启后，存在内存的热缓存就会丢失）。无内存缓存也降低了GC压力，从而对VM—based的语言非常友好（例如Java）。由于producer和cunsumer都是顺序读文件，而且consumer比producer会有一些lag，os的缓存机制会非常有效。  
- 网络优化: 从broker的log file发送消息bytes到连接consumer的socket，需要四次数据复制和两次系统调用。Kafka使用os的sendfile API，直接将数据从file channel转移到socket channel，节约了两次复制和一次系统调用。  
- 无状态broker: 与多数消息系统不同，consumer的消费进度由consumer自己维护而不是broker维护。这大大降低了broker的复杂度，但让删除消息变得tricky，因为broker不知道一条消息是否已经被所有consumer消费。Kafka的策略是如果一条消息在broker中存在了一定时间（例如7天）就会被自动删除。这个策略是非常有效的，因为几乎所有数据都会在这个限制时间内被消费（得益于kafka的性能不会因为数据量增长而降低）  

## 分布式
producer发送一条消息会发到一个随机或者选定的partition。Kafka中有consumer group消费组这个概念，一个消费组中有多个consumer，共同消费订阅的所有topic。消费组之间是隔离的，每个消费组会独立消费自己订阅的消息。消费组内共同消费消息，一条消息只会被消费组中的一个consumer消费。消费组内的consumer可以分布在不同进程或不同机器。Kafka的分布式需要将消息平均分配给不同consumer，而且不引入过多的overhead。  
- 每个消费组中，一个partition的消息会被同一个consumer消费，这避免了分发消息给不同consumer所需要的锁和状态维护。在这样的设计下，consumer只有发生rebalance的时候才会需要分布式调度，这发生的频率是很低的。为了实现负载均衡，kafka需要topic被分成很多份partition，partition数量要多于消费组中consumer的数量。（这解释了为什么kafka要求partition数量必须多于consumer数量，因为如果partition少，会有consumer分配不到partition，变成冗余）
- 去中心化: Kafka没有master节点的概念，而是consumer之间进行分布式调度，这避免了单点故障（master宕机）的问题。为此，Kafka引入了一个高可用的分布式一致性服务Zookeeper。Zookeeper有类似文件系统的API，可以创建path，给path赋值，读path值，删除path，获取path下的子path。此外，一个节点可以设置watcher在一个path上，当path有变化的时候会获得通知。还可以设置临时path（ephemeral），当创造path的client销毁了，path会自动删除。Zookeeper会复制数据到多个节点，保证高可用。
- Kafka在以下场景使用Zookeeper: 感知consumer和broker的添加和删除；当上一个场景发生的时候触发consumer的rebalance；维护消费关系和每个partition的消费offset；当一个新broker或consumer启动后，会把信息注册到Zookeeper上。Broker信息包括host名称，port，topic和partition。Consumer信息包括所属的消费组，订阅的topic。每个消费组在Zookeeper上通过不同的path存储订阅的partition，对应的value是消费这个partition的consumer的id。同时存储了不同partition中上一次消费的offset。这些存储的信息都是临时path，所以当broker或consumer宕机，其注册的信息都会从Zookeeper自动删除。每个consumer都会在这些信息上设置watcher，所以消费组和broker的变动都会通过watcher通知到consumer。
- Rebalance: 当一个consumer新启动或者通过watcher感知到别的consumer/broker变化时，会触发rebalance。Rebalance决定了consumer应该消费哪个partition。Rebalance算法对topic下的partition以消费组中订阅了topic的consumer数为mod作了类似hash的操作，把partition分配给每个consumer。之后会把新的信息注册到Zookeeper，并且新起一个线程根据Zookeeper上存储的offset从partition开始pull数据。由于一个消费组内多个consumer接收到watch通知的时间不完全相同，所以有可能会出现rebalance时分配到的partition仍然被别的consumer占用的情况（别的consumer还没有收到watch通知）。这时consumer会释放所有持有的partition，等待一段时间再重新触发rebalance。一般在重试几次后rebalance都会稳定下来。当新消费组被创建后，Zookeeper上没有对应的offset信息，会根据配置从partition可用的最小或者最大offset开始消费。

## 可靠性保障


# 总结
可以看到，Kafka被设计出来要支持的场景是 **在线场景下实时日志数据的处理** ，这点非常重要，因为Kafka的很多设计都是为了这个场景服务。
- 可扩展：分布式
- 日志数据量大：高吞吐
- 在线实时：低延迟
