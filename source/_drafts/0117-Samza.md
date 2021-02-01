---
title: "1/17 Samza: 可扩展的状态流处理"
categories:
  - Paper Reading Group
date: 2021-01-17 10:12:51
---

### 目标
1. 大状态（local, partitioned)
2. 容错，快速恢复
3. 结合批+流处理

Samza v.s. MapReduce(batch)
Samza(Computation) v.s. Kafka(Storage)

### 流处理管道

服务层 -> Ingestion(**replayable**, 消息系统Kafka，db变化捕获Databus) -> Processing(Samza Jobs) -> Serving(db, return)

### Samza的设计

1. Job: op 的有向图
2. Stream: 无穷的消息流，消息是一个键值对
3: op: transformation, 1:1(map, filter, window, partition), m:1(join, merge), 1:m(split)
4. task: min logical unit, container: min physical unit

4. jobs -> tasks -> containers, input -> partitions, hashing
5. # input partition == # tasks

6. Resource Manager(global?, Apache YARN): resource allocation, monitor, failure detection & recovery
7. Coordinator(per job, Zookeeper): job metadata(#containers, #input src, container plancement, tasks->containers, input partitions->tasks, latest update offset)

### State in Samza

ALT 1: remote store, network overhead

ALT 2: local + full checkpointing, speed down

=> local(incremental) + changelog + reprocessing

1. recovery: replay changelog(rebuild state) -> reprocess/replay msgs after latest offset of changelog
2. at-least-once processing guarantee
3. read-your-write consistency on a per task basis
4. Host Affinity: preferring to place a restarting task on the same physical maachine and try to restore states in disk(store state in a known directory outside of the container)

### Batch + Stream

ALT: Lambda Architecture: parallel fork, high management cost

=> Unified model + Unified api

1. Late Event: reactive approach, processing and fixing previous results when late message arrives(optimize: split into windows?)
2. Reprocessing, reprocessing: switch between(robin hood?) diff inputs -> not interfering real-time traffic, merge job -> prioritize the real-time results

