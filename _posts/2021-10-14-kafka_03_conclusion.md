---
layout: post
title: "重识Kafka-03:总结"
permalink: /kafka_03/conclusion/
---

# 指南
上一篇: [重识Kafka_02:架构设计](https://spikeryang.github.io/kafka_02/design/)  
本篇的内容对应论文的4.kafka在linkedin的落地 5.实验数据 6.总结和未来工作


# 总结
## Kafka在linkedin的落地
- 在linkedin, 每个运行直面用户的服务的DC（数据中心）会配置一个kafka集群。这些服务每秒会产生大量log数据，linkedin使用硬件负载均衡把这些log数据分发到不同的broker。在线的kafka consumer也运行在这个DC里。同时在另一个DC还会配置一个kafka集群做离线数据分析，这个集群的部署位置会靠近Hadoop和数据仓库集群。这个kafka集群的consumer从在线kafka集群pull数据，之后数据加载任务会把这些数据转移到Hadoop和数据仓库进行数据分析，这里的端到端延迟在10s左右。
- 为了追踪数据丢失，每个消息都会带上时间戳和产生它的服务名。每个producer定期产生监控信息，记录了时间窗口内每个topic产生的消息数量，这些监控信息存储在一个单独的topic上。consumer可以用监控信息比对自己收到的消息量检验消息的正确性。
- 加载信息到Hadoop由一种特殊的kafka输入格式实现，这个格式支持MapReduce任务直接从kafka读取数据。

## 实验数据
两台实验机器都是8核16g, 6硬盘，通过1Gb网络连接，一台broker，一台producer/consumer。总共发送1000w条消息，每条200bytes。当发送batch=1的时候，kafka的吞吐率约 5w msg/s，是RabbitMQ、ActiveMQ的两倍。batch=50的时候，kafka的吞吐率达到了惊人的40w msg/s。  
Kafka的惊人性能得益于几个原因，首先producer不需要接等待broker的ack，这大大提升了发送的吞吐率。在50batch的时候，几乎打满了1Gb的网络带宽。这样的优化对于日志收集的场景是合理的，因为必须通过异步发送的方式来避免延迟。但这样无ack的发送方式是没有可靠性保证的，消息会丢失。这样的trade-off对于当前场景可以接受，同时未来kafka也计划解决关键数据的可靠性问题。第二点在于Kafka有更高效的消息存储格式。对每条消息，Kafka只引入了9bytes的额外数据存储，比ActiveMQ的144bytes少了70%。ActiveMQ的消息存储如此臃肿是由于JMS要求的header，还有维护的很多索引结构。最后一点就是batch发送，这大大降低了RPC的开销。  
Kafka消费的吞吐率大约是2w msg/s, 是ActiveMQ和RabbitMQ的4倍。第一个原因是消息存储格式，由于kafka消息更轻量，所以从broker pull的速度会更快。第二是因为kafka不需要自己维护每条消息的投递状态，所以broker上没有写硬盘操作。最后是因为使用了sendFile API，减少了传输开销。  

## 总结和未来工作
再次重温一下，Kafka被设计出来要支持的场景是 **在线场景下实时日志数据的处理** 。 Kafka使用了pull的消费模式，这让consumer可以按照自己的速率消费消息，并且随时回到旧的offset重新消费。针对日志手机场景，Kafka相比传统消息系统大幅提高了吞吐率。Kafka同样支持了分布式部署，保证可扩展性。  
在未来，有几点Kafka希望可以继续完善。
- 为了应对无法恢复的机器故障，Kafka希望在多broker之间引入消息备份replica机制，这可以保证数据的持久性和高可用。replica机制希望可以支持同步和异步发送，在发送延迟和消息可靠性间做trade-off。不同的应用可以根据场景选择数据冗余等级，来获得相应的持久性，可用性和吞吐率。
- 针对流式数据处理，Kafka希望引入流式处理的库。
