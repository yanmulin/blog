---
title: "1/9 Kafka: 分布式日志系统"
categories:
  - Paper Reading Group
date: 2021-01-09 20:12:37
---

目标：
1. 高吞吐
2. 实时
3. 专门日志处理 
4. 分布式，scalable
5. 开源

比较：
1. 杀鸡用牛刀，overkill for log processing
2. 低吞吐
3. 不支持分布式
4. 队列容量小
5. 只支持离线处理(aggregator & analysis 分开运行))
6. 暴露实现细节
7. push model

架构
1. Broker, Topic, Partitions
1. 不考虑Fault Tolerance, 对于一个CG，每个Broker存的P不同
1. Producer用法: 序列化->bytes，批量发布
2. Consumer用法: iterator接口，阻塞模型，流
3. point-to-point model(many/CG)，pub/sub model（1/CG)
4. broker -> partitions

存储
1. topic -> partition -> logical log -> seg files(same size)
2. producer 发布: 追加到seg file末尾 -> flush -> consumer 可用(延迟)
3. consumer pull request: offset + winsize -> consume: offset寻址 -> ack: consumer已接收前面所有消息
4. broker: 有序的offset列表，每一个offset表示seg file第一个消息的offset
5. 顺序读写seg file：在OS的write-thru和read-ahead策略下高效
6. normal: file -> page cache -> app buf -> kernel buf -> socket, sendfile: file -> socket

无状态的Broker：由Consumer维护offset状态
1. 删除：基于时间的SLA，e.g. 7天未访问删除
2. 回退(rewind)：pull model

分布式支持
1. Producer发布到Partition：随机，key，哈希函数
2. Consumer从Partition拉取：CG订阅topic的一个副本，互不干扰
3. #P > #C in CG
4. Partition是并行的单元
5. consumer去中心化，利用Zookeeper管理：Consumer/Broker的添加/删除，Consumer对Partition的所有权，Partition的offset
6. broker registry(ephermal): host, port
7. consumer registry(ephermal): 隶属的CG, 订阅的topics
8. 每个CG有ownership registry(ephermal), offset registry(persistent)
9. 每个Consumer监视broker reg & consumer reg，触发时rebalance
10. 竞争 -> 稍等重试 -> 收敛

Delivery保证
1. 至少一次delivery，重复：Consumer收到消息后宕机，却没有更新offset
2. 单个partition保障消息顺序，多个partition不保障
3. I/O错误 -> CRC -> 清除


Partition Replication
1. P leader handles requests from producers/consumers/followers
2. committed
3. in-sync
