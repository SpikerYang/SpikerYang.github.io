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
在linkedin, 每个

producer发送一条消息会发到一个随机或者选定的partition。Kafka中有consumer group消费组这个概念，一个消费组中有多个consumer，共同消费订阅的所有topic。消费组之间是隔离的，每个消费组会独立消费自己订阅的消息。消费组内共同消费消息，一条消息只会被消费组中的一个consumer消费。消费组内的consumer可以分布在不同进程或不同机器。Kafka的分布式需要将消息平均分配给不同consumer，而且不引入过多的overhead。  
## 可靠性保障
- 通常情况下，kafka只保证 at-least-once 传输（最少一次）。Exactly-once传输需要2pc（两阶段提交），这对于linkedin的场景并不是必须的。对于大多数情况下，消息只会发送到每个消费组一次。但是如果consumer进程在崩溃前没有干净的退出（消费了消息但是没有提交offset到Zookeeper），接管这个partition的新consumer会从旧的offset开始消费一些重复消息。如果你的应用很在意重复消费，可以在应用层根据consumer消费返回的offset或者在消息中加入唯一标识实现重复消息过滤逻辑。这比引入2pc更容易实现。
- Kafka保证一个partition到对应consumer间的消息有序，但是不保证多个partition间的消息有序。为了避免日志污染，Kafka在日志中为每个消息添加了一个CRC（循环冗余校验）。如果broker出现了IO错误，Kafka会运行修复进程删除带有不一致CRC的消息。在消息级别使用CRC也有利于在消息产生或消费后检测网络错误。如果一个broker宕机了，没有消费的消息就不可用了，如果broker底层的存储系统被破坏（机器损坏），任何没有消费的消息就永远丢失了。未来kafka计划支持多broker存储冗余数据来实现内部replica备份。

# 总结
再次重温一下，Kafka被设计出来要支持的场景是 **在线场景下实时日志数据的处理** ，因此在kafka的设计上做了trade-off:
- 牺牲了一些可靠性，一些日志收集场景中不是必须的功能来降低存储和系统的复杂度&提高吞吐率
