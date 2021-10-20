---
title: "[mit6.s081] 笔记 Lab7: Multithreading | 多线程"
date: 2021-10-07 15:48:00 +0800
categories: [Course Notes, MIT6.S081]
tags: [operating system]
---
> 这是我自学 MIT6.S081 操作系统课程的 lab 代码笔记第七篇：Multithreading。此 lab 大致耗时：3小时。  
> 
> 课程地址：[https://pdos.csail.mit.edu/6.S081/2020/schedule.html](https://pdos.csail.mit.edu/6.S081/2020/schedule.html)  
> Lab 地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/thread.html](https://pdos.csail.mit.edu/6.S081/2020/labs/thread.html)  
> 我的代码地址：[https://github.com/Miigon/my-xv6-labs-2020/tree/thread](https://github.com/Miigon/my-xv6-labs-2020/tree/thread)  
> Commits: [https://github.com/Miigon/my-xv6-labs-2020/commits/thread](https://github.com/Miigon/my-xv6-labs-2020/commits/thread)  
> 
> 本文中代码注释是编写博客的时候加入的，原仓库中的代码可能缺乏注释或代码不完全相同。  

# Lab 7: Multithreading

This lab will familiarize you with multithreading. You will implement switching between threads in a user-level threads package, use multiple threads to speed up a program, and implement a barrier.

实现一个用户态的线程库；尝试使用线程来为程序提速；并且尝试实现一个同步屏障

## Uthread: switching between threads (moderate)

补全 uthread.c，完成用户态线程功能的实现。

这里的线程相比现代操作系统中的线程而言，更接近一些语言中的“协程”（coroutine）。原因是这里的“线程”是完全用户态实现的，多个线程也只能运行在一个 CPU 上，并且没有时钟中断来强制执行调度，需要线程函数本身在合适的时候主动 yield 释放 CPU。这样实现起来的线程并不对线程函数透明，所以比起操作系统的线程而言更接近 coroutine。

这个实验其实相当于在用户态重新实现一遍 xv6 kernel 中的 scheduler() 和 swtch() 的功能，所以大多数代码都是可以借鉴的。

uthread_switch.S 中需要实现上下文切换的代码，这里借鉴 swtch.S：

```asm
// uthread_switch.S
	.text

	/*
		 * save the old thread's registers,
		 * restore the new thread's registers.
		 */


// void thread_switch(struct context *old, struct context *new);
	.globl thread_switch
thread_switch:
	sd ra, 0(a0)
	sd sp, 8(a0)
	sd s0, 16(a0)
	sd s1, 24(a0)
	sd s2, 32(a0)
	sd s3, 40(a0)
	sd s4, 48(a0)
	sd s5, 56(a0)
	sd s6, 64(a0)
	sd s7, 72(a0)
	sd s8, 80(a0)
	sd s9, 88(a0)
	sd s10, 96(a0)
	sd s11, 104(a0)

	ld ra, 0(a1)
	ld sp, 8(a1)
	ld s0, 16(a1)
	ld s1, 24(a1)
	ld s2, 32(a1)
	ld s3, 40(a1)
	ld s4, 48(a1)
	ld s5, 56(a1)
	ld s6, 64(a1)
	ld s7, 72(a1)
	ld s8, 80(a1)
	ld s9, 88(a1)
	ld s10, 96(a1)
	ld s11, 104(a1)

	ret    /* return to ra */
```

在调用本函数 uthread_switch() 的过程中，caller-saved registers 已经被调用者保存到栈帧中了，所以这里无需保存这一部分寄存器。

> 引申：内核调度器无论是通过时钟中断进入（usertrap），还是线程自己主动放弃 CPU（sleep、exit），最终都会调用到 yield 进一步调用 swtch。
> 由于上下文切换永远都发生在函数调用的边界（swtch 调用的边界），恢复执行相当于是 swtch 的返回过程，会从堆栈中恢复 caller-saved 的寄存器，
> 所以用于保存上下文的 context 结构体只需保存 callee-saved 寄存器，以及 返回地址 ra、栈指针 sp 即可。恢复后执行到哪里是通过 ra 寄存器来决定的（swtch 末尾的 ret 转跳到 ra）
> 
> 而 trapframe 则不同，一个中断可能在任何地方发生，不仅仅是函数调用边界，也有可能在函数执行中途，所以恢复的时候需要靠 pc 寄存器来定位。
> 并且由于切换位置不一定是函数调用边界，所以几乎所有的寄存器都要保存（无论 caller-saved 还是 callee-saved），才能保证正确的恢复执行。
> 这也是内核代码中 `struct trapframe` 中保存的寄存器比 `struct context` 多得多的原因。
> 
> 另外一个，无论是程序主动 sleep，还是时钟中断，都是通过 trampoline 跳转到内核态 usertrap（保存 trapframe），然后再到达 swtch 保存上下文的。
> 恢复上下文都是恢复到 swtch 返回前（依然是内核态），然后返回跳转回 usertrap，再继续运行直到 usertrapret 跳转到 trampoline 读取 trapframe，并返回用户态。
> 也就是上下文恢复并不是直接恢复到用户态，而是恢复到内核态 swtch 刚执行完的状态。负责恢复用户态执行流的其实是 trampoline 以及 trapframe。

从 proc.h 中借鉴一下 context 结构体，用于保存 ra、sp 以及 callee-saved registers：

```c
// uthread.c
// Saved registers for thread context switches.
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context ctx; // 在 thread 中添加 context 结构体
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;

extern void thread_switch(struct context* old, struct context* new); // 修改 thread_switch 函数声明
```

在 thread_schedule 中调用 thread_switch 进行上下文切换：
```c
// uthread.c
void 
thread_schedule(void)
{
  // ......

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    thread_switch(&t->ctx, &next_thread->ctx); // 切换线程
  } else
    next_thread = 0;
}

```

这里有个小坑是要从 t 切换到 next_thread，不是从 current_thread 切换到 next_thread（因为前面有两句赋值，没错，我在这里眼瞎了卡了一下 QAQ）

再补齐 thread_create：

```c
// uthread.c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  t->ctx.ra = (uint64)func;       // 返回地址
  // thread_switch 的结尾会返回到 ra，从而运行线程代码
  t->ctx.sp = (uint64)&t->stack + (STACK_SIZE - 1);  // 栈指针
  // 将线程的栈指针指向其独立的栈，注意到栈的生长是从高地址到低地址，所以
  // 要将 sp 设置为指向 stack 的最高地址
}
```

添加的部分为设置上下文中 ra 指向的地址为线程函数的地址，这样在第一次调度到该线程，执行到 thread_switch 中的 ret 之后就可以跳转到线程函数从而开始执行了。设置 sp 使得线程拥有自己独有的栈，也就是独立的执行流。

```
$ uthread
thread_a started
thread_b started
thread_c started
thread_c 0
thread_a 0
thread_b 0
thread_c 1
thread_a 1
thread_b 1

......

thread_c 98
thread_a 98
thread_b 98
thread_c 99
thread_a 99
thread_b 99
thread_c: exit after 100
thread_a: exit after 100
thread_b: exit after 100
thread_schedule: no runnable threads
```

### Using threads (moderate)

分析并解决一个哈希表操作的例子内，由于 race-condition 导致的数据丢失的问题。

> Why are there missing keys with 2 threads, but not with 1 thread? Identify a sequence of events with 2 threads that can lead to a key being missing. Submit your sequence with a short explanation in answers-thread.txt

```
[假设键 k1、k2 属于同个 bucket]

thread 1: 尝试设置 k1
thread 1: 发现 k1 不存在，尝试在 bucket 末尾插入 k1
--- scheduler 切换到 thread 2
thread 2: 尝试设置 k2
thread 2: 发现 k2 不存在，尝试在 bucket 末尾插入 k2
thread 2: 分配 entry，在桶末尾插入 k2
--- scheduler 切换回 thread 1
thread 1: 分配 entry，没有意识到 k2 的存在，在其认为的 “桶末尾”（实际为 k2 所处位置）插入 k1

[k1 被插入，但是由于被 k1 覆盖，k2 从桶中消失了，引发了键值丢失]
```

首先先暂时忽略速度，为 put 和 get 操作加锁保证安全：

```c
// ph.c
pthread_mutex_t lock;

int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;
  
  pthread_mutex_init(&lock, NULL);

  // ......
}

static 
void put(int key, int value)
{
   NBUCKET;

  pthread_mutex_lock(&lock);
  
  // ......

  pthread_mutex_unlock(&lock);
}

static struct entry*
get(int key)
{
  $ NBUCKET;

  pthread_mutex_lock(&lock);
  
  // ......

  pthread_mutex_unlock(&lock);

  return e;
}

```

加完这个锁，就可以通过 ph_safe 测试了，编译执行：

```
$ ./ph 1
100000 puts, 4.652 seconds, 21494 puts/second
0: 0 keys missing
100000 gets, 5.098 seconds, 19614 gets/second
$ ./ph 2
100000 puts, 5.224 seconds, 19142 puts/second
0: 0 keys missing
1: 0 keys missing
200000 gets, 10.222 seconds, 19566 gets/second
```

可以发现，多线程执行的版本也不会丢失 key 了，说明加锁成功防止了 race-condition 的出现。

但是仔细观察会发现，加锁后多线程的性能变得比单线程还要低了，虽然不会出现数据丢失，但是失去了多线程并行计算的意义：提升性能。

这里的原因是，我们为整个操作加上了互斥锁，意味着每一时刻只能有一个线程在操作哈希表，这里实际上等同于将哈希表的操作变回单线程了，又由于锁操作（加锁、解锁、锁竞争）是有开销的，所以性能甚至不如单线程版本。

这里的优化思路，也是多线程效率的一个常见的优化思路，就是降低锁的粒度。由于哈希表中，不同的 bucket 是互不影响的，一个 bucket 处于修改未完全的状态并不影响 put 和 get 对其他 bucket 的操作，所以实际上只需要确保两个线程不会同时操作同一个 bucket 即可，并不需要确保不会同时操作整个哈希表。

所以可以将加锁的粒度，从整个哈希表一个锁降低到每个 bucket 一个锁。


```c
// ph.c
pthread_mutex_t locks;

int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;
  
  for(int i=0;i<NBUCKET;i++) {
    pthread_mutex_init(&locks[i], NULL); 
  }

  // ......
}

static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  pthread_mutex_lock(&locks[i]);
  
  // ......

  pthread_mutex_unlock(&locks[i]);
}

static struct entry*
get(int key)
{
  int i = key % NBUCKET;

  pthread_mutex_lock(&locks[i]);
  
  // ......

  pthread_mutex_unlock(&locks[i]);

  return e;
}

```

在这样修改后，编译执行：

```
$ ./ph 1
100000 puts, 4.940 seconds, 20241 puts/second
0: 0 keys missing
100000 gets, 4.934 seconds, 20267 gets/second
$ ./ph 2
100000 puts, 3.489 seconds, 28658 puts/second
0: 0 keys missing
1: 0 keys missing
200000 gets, 6.104 seconds, 32766 gets/second
$ ./ph 4
100000 puts, 1.881 seconds, 53169 puts/second
0: 0 keys missing
3: 0 keys missing
2: 0 keys missing
1: 0 keys missing
400000 gets, 7.376 seconds, 54229 gets/second
```

可以看到，多线程版本的性能有了显著提升（虽然由于锁开销，依然达不到理想的 `单线程速度 * 线程数` 那么快），并且依然没有 missing key。

此时再运行 grade，就可以通过 ph_fast 测试了。

### Barrier (moderate)

利用 pthread 提供的条件变量方法，实现[同步屏障机制](https://en.wikipedia.org/wiki/Barrier_(computer_science))。

```c
// barrier.c
static void 
barrier()
{
  pthread_mutex_lock(&bstate.barrier_mutex);
  if(++bstate.nthread < nthread) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  } else {
    bstate.nthread = 0;
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

线程进入同步屏障 barrier 时，将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数。  
如果未达到，则进入睡眠，等待其他线程。  
如果已经达到，则唤醒所有在 barrier 中等待的线程，所有线程继续执行；屏障轮数 + 1；

「将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数」这一步并不是原子操作，并且这一步和后面的两种情况中的操作「睡眠」和「唤醒」之间也不是原子的，如果在这里发生 race-condition，则会导致出现 「lost wake-up 问题」（线程 1 即将睡眠前，线程 2 调用了唤醒，然后线程 1 才进入睡眠，导致线程 1 本该被唤醒而没被唤醒，详见 [xv6 book](https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf) 中的第 72 页，Sleep and wakeup）

解决方法是，「屏障的线程数量增加 1；判断是否已经达到总线程数；进入睡眠」这三步必须原子。所以使用一个互斥锁 barrier_mutex 来保护这一部分代码。pthread_cond_wait 会在进入睡眠的时候原子性的释放 barrier_mutex，从而允许后续线程进入 barrier，防止死锁。

执行测试：
```
$ ./barrier 1
OK; passed
$ ./barrier 2
OK; passed
$ ./barrier 4
OK; passed
```
