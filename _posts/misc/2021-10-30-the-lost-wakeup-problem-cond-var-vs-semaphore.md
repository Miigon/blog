---
title: "杂记随笔：唤醒丢失问题 & 条件变量 vs 信号量"
date: 2021-10-30 08:38:00 +0800
categories: [Misc, Notes]
tags: [operating system]
---

## Scenario

考虑 producer-consumer 同步模式中的 receiver:
```python
receive(bb):
	acquire(bb.lock)
	while bb.out >= bb.in:
		release(bb.lock)		# release lock before sleep
								# so other threads can run
		wait(bb.has_message) 	# wait on conditional variable
		acquire(bb.lock)		# reacquire
	message <- bb.buf[bb.out mod N]
	bb.out <= bb.out + 1
	release(bb.lock)
	notify(bb.has_space)
	return message
```
在没有新消息进入的时候，receiver 应该放弃共享缓冲区的锁，然后进入睡眠等待 sender 唤醒。
然而上述代码的问题在于，「放弃缓冲区锁」和「进入睡眠」不是一步原子操作，而是独立的两步操作。

## Problem

在 receiver 放弃共享缓冲区锁（`release(bb.lock)`）之后，但是在进入睡眠（`wait(bb.has_message)` ）之前，另一个 sender 有可能在这个间隙中发送消息。

在 receiver 进入睡眠之前，sender 会看到**没有接收者正在等待 has_message**，于是该 receiver 并不会得到消息通知。sender 发送完成后，receiver 才进入睡眠，最好的情况下需要等待下一次 sender 唤醒才能被唤醒，最差情况下永远都不会被唤醒。（deadlock）

同样在 receive 从睡眠中唤醒之后以及重新获取锁之前，并发的 sender 也同样可能发送消息，这一部分消息的通知也无法被 receiver 收到。

该问题被称为 “lost wakeup problem” 或 “lost notify problem”，可译为“唤醒丢失”

> lost wakeup 检测方法：if(receiver 收到通知的次数 < sender 发送通知的次数)

## Solution: atomic release-wait-reacquire

问题出在 `release(bb.lock)` 与 `wait(bb.has_message)` 之间留出来的短暂间隙。解决方法是由操作系统提供一个「原子性释放锁与进入等待」机制，以及「唤醒后原子性重新获得锁」的机制（release-wait-acquire）。

也就是，将这三行代码合成一个原子操作：
```python
release(bb.lock)		# release lock before sleep
						# so other threads can run
wait(bb.has_message) 	# wait on conditional variable
acquire(bb.lock)		# reacquire
```

### monitor (condition variable)
在 POSIX 上，这个机制为 pthread condition variable，包含如下操作：

```python
# atomic [release mutex `m`, sleep on `c`, reacquire mutex after wakeup]
pthread_cond_wait(c, m)

pthread_cond_signal(c)		# wake up 1 thread sleeping on `c`
pthread_cond_broadcast(c)	# wake up all threads sleeping on `c`
```

pthread_cond_wait 提供了原子性的「释放互斥锁—进入睡眠—在唤醒后重新获得锁」操作。

即改为：
```python
receive(bb):
	acquire(bb.lock);
	while bb.out >= bb.in:
		pthread_cond_wait(bb.has_message, bb.lock)
	message <- bb.buf[bb.out mod N]
	bb.out <= bb.out + 1
	release(bb.lock)
	pthread_cond_signal(bb.has_space)
	return message
```

### semaphore
另一个解决思路是使用信号量 semaphore。只适用于能够抽象为「某个整数是否大于 0」的 condition（例如剩余缓冲区数量、剩余消息数量）。

```
sem_wait(s)
sem_post(s)
```

检查是否需要 sleep 的逻辑，和进入睡眠的逻辑一起被包含在了 P 原子操作中（相比于 monitor/condition variable 的由用户程序自己做检测）。

P 原子操作包含了整个「获得互斥锁—判断资源数量—**释放互斥锁—进入睡眠—在唤醒后重新获得锁**」的过程，所以我们所需要的「释放互斥锁—进入睡眠—在唤醒后重新获得锁」过程自然也是原子性的。

### differences between the two

**可以将 semaphore 看作「condition 为`某个整数 > 0` 的 condition variable」**。

semaphore 适合在**等待条件可以用一个整数描述**的时候使用。条件变量的维护工作由 P（wait）、V（signal） 原子操作完成。

condition variable 则将判断等待条件的任务交给了用户程序，提供了更大的自由度和灵活性。可以用来等待一些**不可以用「整数>0」描述的条件变量**，例如网络事件和同步屏障（需要等待整数 = 0 ，信号量为等待整数 > 0）（[s081-lab7-multithreading-barrier](https://blog.miigon.net/posts/s081-lab7-multithreading/#barrier-moderate)）。

小细节：
* 对于 semaphore 来说，signal 操作在没有进程正在等待的时候，并不会丢失，而是会被记录为整数+1
* 对于 condition variable，signal 操作在没有进程正在等待的时候，会丢失。
* semaphore 使用了内部的互斥锁保证原子性，condition variable 使用了外部传入的互斥锁保证原子性

> 可以使用「维护一个整数 i + 等待「i > 0」的 condition variable」来实现 semaphore


References:
* https://en.wikipedia.org/wiki/Monitor_(synchronization)#Condition_variables
* https://en.wikipedia.org/wiki/Producer–consumer_problem
