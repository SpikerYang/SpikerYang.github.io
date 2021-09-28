---
layout: post
title: "重识Kafka-01:诞生背景"
permalink: /kafka_01/background/
---

# 指南
本篇的内容对应论文的摘要, 1.介绍, 2.相关工作

# 摘要
设计者对Kafka的描述：  
- 用于低延迟地收集&&传输大容量日志数据的消息系统
- 同时适合在线和离线的消息消费
- 引入了一些非传统但是有用的设计使系统高效，scalable(可扩展)

# 介绍 & 相关工作
设计者对日志收集场景的分析：  
- 数据量大
- Facebook的Scribe等消息系统只支持离线分析场景
- LinkedIn需要支持秒级延迟的在线分析场景
- 因此Kafka诞生了，结合了传统消息系统和日志收集器的优点。一方面Kafka是分布式的，scalable，有高throughput（吞吐率）；另一方面Kafka提供了接近消息系统的API，并且支持实时消息处理。
