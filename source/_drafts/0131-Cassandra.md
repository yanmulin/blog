---
title: "Cassandra: 分布式高可用的NoSQL数据库"
categories:
  - Paper Reading Group
date: 2021-01-31 10:56:47
---

BigTable + Dynamo

### 特征

1. NoSQL
2. 分布式高可用（容错）
3. BigTable
4. 读写高吞吐

### 数据模型

1. BigTable
2. replica级别的原子操作
3. 根据key/timestamp排序（time-series data, real-time data)

### 系统架构

1. Partition: Consistent Hashing
2. Replication: key -> coordinator -> replicas, ZooKeeper -> leader -range-> new nodes
3. Membership: Gossip-based, Accrual Failure Detector
4. Local Persisstence:
  * in-memory data structure, commit log, data file + column indices + bloom filter
  * Write: commit log -> in-mem data structure -> size threshold, data file
  * Read: in-mem data structure -> bloom filter -> indices (from newest to oldest)
  * Compaction process
5. SEDA architecture

### Use Case

Facebook Inbox Search

* term search
* interaction