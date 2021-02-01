---
title: Unix迷失的唤醒
categories:
  - unclassified
date: 2020-12-14 11:59:43
---

[这篇论文](https://www.usenix.org/legacy/publications/compsystems/1990/sum_ruane.pdf)介绍了Unix内核如何为多核处理器设计基于事件的sleep lock，并利用自旋锁解决唤醒时的竞争问题。这篇论文分析了内核中许多潜在竞争问题，结合Model Checking和Spin，十分值得琢磨且有趣。

### Sleep Lock 与唤醒竞争

锁的获取与释放在Unix内核中被当作事件处理：获取锁时，进程检查资源占用情况，如果已被占用，则把进程放入等待队列，接着睡眠等待；释放锁时，把所有等待队列中的进程唤醒，令他们争夺资源，只有一个进程可以争夺到资源，于是其他进程继续睡眠等待。

```cpp
// Process A
while (r->lock)
  sleep(r);
r->lock = 1;
```

```cpp
// Process B
r->lock = 0;
wakeup(r);
```

这里存在一个竞争问题。假设初始由进程B占用`r->lock`，当进程A运行到第2-3行间，定时器中断来临，切换到进程B运行。接着进程B释放锁`r->lock = 0`，执行唤醒`wakeup`，但这时进程A还没有睡眠。等轮到进程A运行并陷入睡眠时，进程A已经没有机会再被唤醒了。这里产生的唤醒竞争，是本文要解决的核心问题。

### 解决方法

由于竞争产生的根源是中断导致的上下文切换，禁用中断至少可以为单核处理器解决问题。

```cpp
// Process A
interruption_disable();
while (r->lock)
  sleep(r);
r->lock = 1;
interruption_endable();
```

```cpp
// Process B
interruption_disable();
r->lock = 0;
wakeup(r);
interruption_endable();
```

尽管禁用了中断，并发在多核处理器仍然存在。这里引入自旋锁解决问题。自旋锁可以提供处理器级别的同步，用于短期的互斥访问。

```cpp
// Process A
spinlock(&r->lk);
while (r->lock)
  sleepl(r, &&r->lk);
r->lock = 1;
freelock(&r->lk);
```

```cpp
// Process B
r->lock = 0;
waitlock(&r->lk);
wakeup(r);
```

这里`waitlock`只等待自旋锁被释放，而不获取锁。使用`waitlock`而非`spinlock`可以减少争夺锁带来的性能下降。进程B在释放`r->lock`后等待自旋锁的释放，确保进程A要么陷入睡眠状态，要么跳过`while`循环。睡眠状态的进程不能持有自旋锁，所以自旋锁被传入`sleepl`，陷入睡眠前释放，被唤醒后重新获取。


### 引入want减少唤醒次数

`wakeup`实际上将所有等待队列中的进程唤醒，带来很大的上下文切换开销，引入一个`want`变量避免不必要的唤醒。但是也引入了新的竞争问题。

```cpp
// Process A
spinlock(&r->lk);
while (r->lock) {
  r->want = 1;
  sleepl(r, &r->lk);
}
r->lock = 1;
freelock(&r->lk);
```

```cpp
// Process B
r->lock = 0;
waitlock(&r->lk);
if (r->want) {
  r->want = 0;
  waitlock(&r->lk);
  wakeup(r);
}
```

一定需要两个`waitlock`，第一个用于保护`want`必须在测试前被置一，第二个确保不存在唤醒竞争。一个有点难的问题：如果不存在第二个`waitlock`，竞争如何触发？假设进程A刚刚被唤醒，而进程B正在释放资源。由于进程A还没尝试获得自旋锁，进程B在第一个`waitlock`没有遇到障碍并通过了`r->want`的测试（其他进程设置了`r->want`）。接着进程A从`sleepl`返回，测试`r->lock`，可资源已经被其他进程抢走，于是进程A重新进入睡眠。在这个情况下，如果没有第二个`waitlock`，唤醒竞争仍然存在。


### 条件获取Sleep Lock

Unix内核中有这样的情况：尽管进程抢到了锁，它也不一定获取这个锁。比如文件系统的空闲块，以链表的形式存储在内存中。进程只在链表满的时候才真正获取保护链表的锁。这个例子中，自旋锁还保护链表的操作。

```cpp
spinlock(&fp->lk);
while (fp->lock)
  sleepl(fp, &fp->lk);
if (isfull(fp->free)) {
  fp->lock = 1;
  freelock(&fp->lk);
  write fp->free into disk
  set fp->free to empty
  spinlock(&fp->lk);
  fp->lock = 0;
  wakeup(fp);
}
append block being freed to fp->free
freelock(&fp->lk);
```

可以考虑下面的唤醒实现为什么不可行。

```cpp
fp->lock = 0;
spinlock(&fp->lk);
wakeup(fp);
```

利用Spin可以验证，这个唤醒实现的问题在于可能导致链表溢出。比如进程在释放`fp->lock`后被打断，链表被填满，然后进程恢复并继续运行到13行，导致链表溢出。

这里同样可以引入`want`变量减少唤醒次数。

### 跨进程消息队列的Sleep Lock

消息队列有三个锁：队列锁，消息锁和自旋锁。队列锁用于操作队列，消息锁用于消费者等待空队列，自旋锁用于保障处理器级别的同步。其中自旋锁是IPC全局共享的。

```cpp
// Sender
spinlock(&msg_lk);
while (qp->lock)
  sleepl(qp, &msg_lk);
qp->lock = 1;
freelock(qp->lock);
/* Enqueue a message */
wakeup(qp->qnum);
qp->lock = 0;
waitlock(&msg_lk);
wakeup(qp->qnum);
```

```cpp
// Receiver
loop:
spinlock(&msg_lk);
while (qp->lock)
  sleepl(qp, &msg_lk);
qp->lock = 1;
freelock(qp->lock);
/* if no message to read */
spinlock(&msg_lk);
qp->lock = 0;
wakeup(qp);
sleepl(&qp->qnum, &msg_lk);
freelock(&msg_lk);
goto loop
```

首先，Receiver要确保在释放`qp->lock`前获取自旋锁，否则`sleepl(&qp->qnum, &msg_lk)`对应的唤醒可能丢失。其次，`wakeup(qp->qnum)`前面不需要等待自旋锁，因为此时Sender已经拥有`qp->lock`，Receiver要么正在等待`qp->qnum`，要么正在等待`qp->lock`，不可能产生唤醒竞争。


### 结论

通过几个唤醒竞争的例子，可以一窥并发编程与竞争分析的难点。正如信息安全学，原语并不能保证系统的安全性。程序的线程安全隐患，往往在于非常隐蔽且微妙的地方。