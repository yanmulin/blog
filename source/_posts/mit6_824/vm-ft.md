---
title: "VMWare虚拟机的容错设计"
categories: 
  - MIT 8.624 Papers
date: 2021-01-31 15:23:37
---

这篇论文描述了一个机器级的容错架构，以很小的网络带宽消耗实现了一个可容错的商用虚拟机。

这个架构的设计思路是把机器抽象为一台状态机，如果primary和backup接受完全相同的输入和事件，那么两者最终会抵达相同的状态。所以只要将primary的所有输入和非确定性事件(non-deterministic operations/events)原封不动的通过logging channel发送给backup，那么在primary宕机时，backup也能恢复到与primary相同的状态。backup通过log entries恢复状态的过程叫做**非确定性重放(non-deterministic replay)**。

### 非确定性重放

非确定性重放要求primary把所有输入和非确定性事件发送给backup。输入包括但不限于收到的网络包，磁盘读取，键鼠等设备输入等；非确定性事件包括虚拟的中断，产生随机数，和读取CPU时钟等。多亏虚拟机，输入和非确定性事件才可以被捕获与记录。重放的过程如下：

1. 捕获到输入或非确定性事件时，primary通过logging channel发送给backup；
2. backup收到并将消息入队(并持久化）后，才给primary发送ack回复；
3. primary收到来自backup的ack回复后，才回复client；
4. backup不定时地消耗队列中的消息。

消息内容需要包括事件发生时primary的指令序号，因为backup也需要在相同的指令位置触发事件。队列为空时，backup会停止执行等待primary发来新的消息；队列满时，primary停止执行等待backup消耗消息。

Primary在收到backup的ack才回复client是为了满足**输出法则(Output Rule)**。如果primary在收到ack前回复client，且primary在回复client后宕机，那么不能确保backup收到这条消息，也就不能恢复到primary宕机前的状态。在目前设计下，如果primary在收到backup的ack后回复client，接着发生宕机，那么backup提升为primary后不能确定以前的primary有无给client发送回复。为了保险起见，backup都会给client重新发送一次回复。所以系统只能保证请求被处理“至少一次”，而且需要引入去重(deduplication)的机制。

### 错误检测与恢复

论文采用了简单的Fail-stop的错误模型。如果没有在限定时间内收到回复，尽管真正的原因可能是网络延时，也假设目标宕机。

VMWare的错误检测结合了UDP心跳与logging channel监听的两种方式。因为虚拟机定期会产生时钟中断，运行正常的虚拟机会周期性的通过logging channel发送消息。如果primary宕机，backup通过重放队列中的日志，恢复到primary宕机前的状态。

如果发生裂脑错误(split-brain)，VMWare在CAP中优先保证一致性，在这个情况下primary和backup只有一个能够存活。为了确定哪一个能够存活，需要引入一个原子的test-and-set服务器。如果检测到裂脑，primary和backup会向test-and-set服务器请求操作，得到回复为0的一方可以存活并提升为primary，另外一方则自动停机。

### 局限

该论文假设primary和backup共享同一个磁盘，这个假设消除了持久化层面一致性的影响。另外，截止于论文发表的时间，这个架构的虚拟机只支持模拟x86单核处理器。因为多核的并行执行引入了更复杂的非确定性。

