# [Paper Review] Building a database on S3

> 原文：[Building a database on S3](https://dl.acm.org/doi/pdf/10.1145/1376616.1376645) SIGMOD '08: Proceedings of the 2008 ACM SIGMOD international conference on Management of data

## Overview

- 探索s3上建立支持小对象和频繁更新的可能性和挑战
- 说明如何在s3上实现B-tree
- 提出如何在s3上实现不同级别的一致性协议
- 使用TPC-W测试在不同的一致性级别下使用s3的时间以及成本
- 主要面向 Web-based 应用的一致性设计，即一致性要求不高
- 论文说明了在s3上建立数据库系统和传统情况下有众多相似之处

s3局限性

- s3与本地盘相比速度慢
- s3廉价且高可用，但是牺牲了一致性（s3仅仅保证了最终一致性）

## AWS

## S3

`Simple Storage System` 支持变长对象存储（最小1字节，最大5GB），对象的元数据（最大4KB）可以与对象相关联但是也能与对象之间独立地进行修改与读取

s3的延迟与带宽测试结果：

| Page Size(KB) | Resp. Time(secs) | Bandwith(KB/secs) |
|:--------|:-------:|--------:|
| 10 | 0.14 | 71.4 |
| 100 | 0.45 | 222.2 |
| 1000 | 3.87 | 258.4 |

s3的读请求延迟相比于本地盘要多出2-3个数量级，但是在吞吐率上要比本地盘好很多，而且s3的吞吐率与并发请求的用户数量无关；从带宽上看，小对象不能的读取很难达到能够接受的带宽，s3对大对象的读取更加友好（通常将小对象集合到一个大page中进行传输）

## SQS

`Simple Queueing System` 是一个不保证顺序的消息队列服务。

SQS的请求RTT测试结果：

| Operations | Time(secs) |
|:--------|-------:|
| send | 0.31 |
| receive | 0.16 |
| delete | 0.16 |

## EC2

`Elastic Computing Cloud` 

## 架构


对 client 满足：

- 允许随时发生崩溃
- 读写请求能在稳定的时间内完成
- client 之间不会互相阻塞

带来的代价是s3较弱的一致性：客户端需要等待不确定的时间才能看到其他某个client的更新

存储层是AWS的S3.
虚拟数仓层（virtual warehouse）负责在vm集群上执行query。
云服务层，包括了管理VW、query、事务的服务，以及管理元数据的服务。
