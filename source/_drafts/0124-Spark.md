---
title: "01/24 Spark: 支持工作集的集群计算"
categories:
  - Paper Reading Group
date: 2021-01-24 13:10:29
---

## 与Hadoop的比较

1. 迭代计算：计算过程重复使用到同一个数据集，Hadoop作为独立的job，需要重新从磁盘读取数据
2. 交互式

### 编程模型

1. RDD: 只读，partition -> node，容错，lazy
2. 并行操作：reduce，collect，foreach
3. 共享变量：广播变量（只读，所有节点共用），累加器（零，加，满足结合律）=> fault tolerance
4. Transformation v.s. Action

### 实现

#### RDD

1. RDD接口
  * getPartitions
  * getIterator(partition)
  * getPreferredLocation(partition)
2. 不同RDD类型采用不同的实现
  * MappedDataset
  * CachedDataset
  * HdfsTextFile
3. 数据族谱
3. 容错：重新计算
4. shipping task: closure

#### 共享变量

1. 广播变量：数据持久化在shared fs(hdfs)，将路径序列化
2. 累加器：序列化unique id & zero，update once