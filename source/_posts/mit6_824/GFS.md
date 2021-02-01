---
title: Google File System
date: "2020/11/27"
layout: post
categories: 
- MIT 8.624 Papers
---

GFS是谷歌的三驾马车之一，也是分布式系统学习过程躲避不开的经典案例。经典论文常读常新，本文不追求面面俱到，只总结对我有启发的要点。

### 架构

一个GFS集群包含一个Master节点和若干chunkserver节点，节点进程在Linux用户态运行。Master节点维护文件系统的元数据，而文件数据以chunk为基本单位存储在chunkserver节点中。chunk由全局唯一的句柄（handle）识别，Master管理文件名到chunk句柄的映射，以及chunk句柄到chunkserver地址的映射。

Master中某些元数据存储在内存和硬盘中，比如文件名到chunk句柄的映射。Master采用logging和checkpoint的持久化策略，因为logging是顺序写入磁盘，相比数据库（B-tree）随机写入的速度提高不少。但有些元数据只存在内存中，比如chunk句柄到chunkserver的映射。由于内存的易失性，Master启动时需要主动向chunkserver请求所有chunk信息，往后周期性的心跳也被利用来更新chunk句柄到chunkserver的映射。

### 读操作

读操作非常简单：

客户端 -> Master：文件名，偏移量
Master -> 客户端：chunk句柄，目标chunkserver地址列表（结果被缓存）
客户端 -> 最近的chunkserver：读取请求
chunkserver -> 客户端：数据
一个GFS集群所有节点（包括客户端）都位于同一个机房中，所以离客户端最近的chunkserver可以通过IP地址计算得到。

### 数据版本与租约

GFS利用数据版本（Version Number）来区分过期的数据。Master记录了每个chunk的最新数据版本。每当客户端需要chunkserver地址来访问某个chunk时，Master只返回有最新版本的chunkserver。

租约（Lease）是用来管理数据版本的机制。一份租约在chunkserver中选择一个primary，其余的成为secondary。租约有有效期：当租约过期，Master需要达成新的租约，并更新版本号。

Master达成租约的过程如下：

根据请求中的chunk句柄，获得当前最新的数据版本号；
筛选出拥有该chunk最新版本的chunkserver；
在筛选结果中选择一个chunkserver作为primary，其余作为secondary；
更新版本号，并通知primary和secondaries最新的版本号；
持久化最新的版本号。
对于某个chunk，整个GFS只允许存在一个primary。租约的有效期保障了这一点。假设租约是无限期的，如果发生“裂脑”错误且Master没有收到primary的心跳，于是判定primary挂掉了，重新指定一个primary。但实际上原来的primary仍然正常运行并处理客户端的请求，导致严重的不一致错误。如果租约只在一定时间范围有效，即使Master没有收到primary的心跳，也应该等待当前租约过期且原来的primary失去资格，才重新指定primary，从而确保只存在一个primary。

### 写操作

GFS的写操作以chunk为单位，如果客户端写入超过chunk大小的数据，或者写入范围跨越多个chunk，则会被分割为多次写操作。写操作为了提高网络带宽的利用率，把过程分为控制流和数据流。控制流负责向Master和primary传递指令，数据流将待写入的数据传送给目标chunkserver的缓冲区。完整的写入过程如下：

客户端 -> Master：租约（若不存在则创建）；
Master -> 客户端：租约中的primary和secondaries（结果被缓存）；
客户端 -> primary, secondaries缓冲区：数据，未真正写入；
客户端 -> primary：写指令；
primary -> secondaries：数据从缓冲区写入硬盘；
如果上一步写入全部成功，primary回复成功；否则回复失败，客户端重试。
追加记录操作

Google应用大部分写入操作都是追加记录（Append Record），所以追加记录操作对性能要求非常高。除了写入的位置由Master计算出来，其他写入的过程与随机写基本一样，这里不赘述。追加记录操作可以确保并发写入时数据被至少追加一次，“至少”说明数据可能存在重复的状况，副本间可能存在不一致的问题。比如，在写入的第5步部分chunkserver发生错误，而其他chunkserver成功写入，于是客户端重试成功后，某些chunkserver的文件中存有重复的数据。

由于GFS的上层应用（如MapReduce）对性能敏感，而对重复数据不敏感，所以为了性能放弃一致性是合理的。这里又是一个典型例子，根据上层应用的特点，适当放松一致性要求。再者，重复数据可以通过全局唯一的ID去重容易地解决。

### 其他特性

由于硬盘存在翻位错误，chunkserver利用checksum校验保障数据完整性（Data Integrity）。若数据校验失败则使用其他副本的数据。

GFS提供快照（Snapshot）功能。快照采用写时复制（copy-on-write）的策略，所以Master每个chunk句柄拥有一个计数变量来记录被快照引用的次数。当写入引用次数大于1的chunk，数据才真正进行被复制。

Master的影子副本存放较老的数据，只读，为要求较宽松的客户端提供服务，降低真正Master的压力。

### 缺陷

GFS的缺陷在于Master是单一故障点。单一Master有限的内存大小和请求处理速度使其容易成为系统瓶颈。尽管Master有备份，但Master的故障恢复仍需要人工干预，效率较低。