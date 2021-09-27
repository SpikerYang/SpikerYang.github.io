layout: post
title: "重识Kafka-01:诞生背景"
permalink: /kafka_01/background/

# 指南
本篇的内容对应论文的摘要, 1.介绍, 2.相关工作

# 摘要
设计者对Kafka的描述：  
- 用于低延迟地收集&&传输大容量日志数据的消息系统
- 同时适合在线和离线的消息消费
- 引入了一些非传统但是有用的设计使系统高效，scalable(可扩展)

# 介绍
设计者对日志收集场景的分析：  
- 数据量大
- Facebook的Scribe等消息系统支持离线分析场景
- LinkedIn需要支持秒级延迟的在线分析场景
