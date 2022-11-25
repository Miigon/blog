---
title: "Linux 是否有 zombie thread？从glibc和内核源码探究"
date: 2022-11-23 17:47:00 +0800
categories: [Linux]
tags: [Chinese, linux]
---

系统编程课上遇到的一个问题：Linux下，如果一个 `pthread_create` 创建的线程没有被 `pthread_join` 回收，是否会和僵尸进程一样，产生“僵尸**线程**”？

# 猜想
 
## 僵尸进程

对于进程与子进程来说，如果子进程退出了，但是父进程不对子进程进行 reap （即使用 wait/waitpid 对子进程进行回收），则子进程的 PCB（内核中的 task_struct）依然会保留，用于记录返回状态直到父进程获取，并且状态将被设置成 ZOMBIE，即产生“僵尸线程”。
```c
#include <unistd.h>

int main()
{
    if(fork() == 0) {
        return 0; // child exits immediately
    }
    while(1); // parent loops
}
```

运行后可以看到子进程 607727 的状态为 Zombie，并且在最后有 `<defunct>` 标志。
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
miigon    607726 97.5  0.0   2640   988 pts/0    R+   21:55   0:28 ./child
miigon    607727  0.0  0.0      0     0 pts/0    Z+   21:55   0:00 [child] <defunct>
```

## 僵尸*线程*？

```c
#include <stdlib.h>
#include <pthread.h>

void *child_thread(void *args) {
    // child thread exits immediately.
    return (void*)666;
}

int main() {
    pthread_t t1;

    pthread_create(&t1, NULL, child_thread, NULL);

    while(1);
    // parent and child never join.
    // pthread_join(t1, NULL);
}
```

子线程启动后马上返回，而父线程无限循环，并且不 join 子线程，不检查其返回状态。

Linux 内核中（至少在调度上）并不区分线程和进程，都视为 task，故合理猜想：可能这里的 pthread_create 和 pthread_join 也可以类比 fork 和 wait，如果一个线程被创建后，不进行 pthread_join，那在子线程执行结束后，可能子线程也会进入 Zombie 状态，直至被父线程回收？（猜想）

# 验证

对上述猜想进行验证，编译运行上述线程代码：
```
$ gcc pt.c -o pt -lpthread -g
$ ./pt
```

如果我们的猜想正确，当查看 pt 的所有线程的时候，理论上应该可以看到一个主线程，还有一个 defunct 状态的子线程 task。

使用 ps 检查 pt 的所有线程：
```
$ ps -T -C pt
    PID    SPID TTY          TIME CMD
 610281  610281 pts/1    00:00:09 pt
```

发现只有一个主线程（`PID == SPID`），没有观察到 defunct 状态的子线程，子线程退出后虽然主线程没有 pthread_join 读取其返回值，但是子线程 pid/tid 依然被回收了，并没有进入僵尸状态。

验证一下如果把子线程函数换成死循环，运行后可以观察到子线程存在，说明测试方法没有问题，排除子线程没有创建成功或者观测方法有误的可能性：
```c
void *child_thread(void *args) {
    while(1);
}
```

```
$ ps -T -C pt
    PID    SPID TTY          TIME CMD
 610762  610762 pts/1    00:00:05 pt
 610762  610763 pts/1    00:00:05 pt
```

说明我们的猜想是不准确的，并没有观察到子线程退出后变为僵尸状态。

# 探究

由于已知在 Linux 上，创建线程和创建进程实际上走的是同一套机制，本质上都是 fork/clone，只是调用者指定的资源共享程度不同，所以差异出现的诱因只能是位于 fork/clone 的调用者，即位于 pthread 的代码中。

pthread 在 Linux 上一般是由 libc 实现的，最常见的 libc 是 glibc（另一个 Linux 上常用的 libc 的例子是 musl，更轻量，不展开）。glibc 的 pthread 实现叫做 NPTL（替换掉之前的远古实现叫 LinuxThreads，也不展开），可以在 <https://codebrowser.dev/glibc/glibc/nptl/> 很方便地在线浏览相关代码。

> 本文环境 ubuntuserver 22.04.1 + linux5.15.0 + glibc2.35；所有源代码文件以这些版本为准。
> 如果发现 glibc/NPTL 部分代码的锁进很乱，那是由于原来的代码就是这么锁进的，不是文章格式化错误。

## 线程等待 pthread_join()

首先检查 pthread_join 的源码，因为根据我们的猜想，如果是会产生“僵尸线程”的话，pthread_join 要回收这个“僵尸线程”，必然要调用 wait/waitpid 系的系统调用。

<https://codebrowser.dev/glibc/glibc/nptl/pthread_join.c.html>
<https://codebrowser.dev/glibc/glibc/nptl/pthread_join_common.c.html>
```c
// pthread_join.c:21
int
___pthread_join (pthread_t threadid, void **thread_return)
{
  return __pthread_clockjoin_ex (threadid, thread_return, 0 /* Ignored */,
				 NULL, true);
}
```

核心部分：

```c
// pthread_join_common.c:35
int
__pthread_clockjoin_ex (pthread_t threadid, void **thread_return,
                        clockid_t clockid,
                        const struct __timespec64 *abstime, bool block)
{
  struct pthread *pd = (struct pthread *) threadid;
  
  // ......

  if (block) // true
    {
	  // 等待线程执行完成
      pthread_cleanup_push (cleanup, &pd->joinid);

      pid_t tid;
      while ((tid = atomic_load_acquire (&pd->tid)) != 0) // 获取锁
        {
	  int ret = __futex_abstimed_wait_cancelable64 ( // 通过等待一个 futex 来等待线程执行完成，只是 futex syscall 的封装，内部并没有调用 wait/waitpid
	    (unsigned int *) &pd->tid, tid, clockid, abstime, LLL_SHARED);
	  if (ret == ETIMEDOUT || ret == EOVERFLOW)
	    {
	      result = ret;
	      break;
	    }
	}

      pthread_cleanup_pop (0);
    }

  void *pd_result = pd->result; // 获取线程的返回值，说明线程已经执行完成
  if (__glibc_likely (result == 0)) // 等待成功
    {
      /* We mark the thread as terminated and as joined.  */
      pd->tid = -1;

      /* Store the return value if the caller is interested.  */
      if (thread_return != NULL)
	*thread_return = pd_result;

      /* Free the TCB.  */
      __nptl_free_tcb (pd); // 释放 TCB，即 pthread 结构体
    }

  // ......
}

```

通过 JOIN 的这部分关键代码，可以推测出这几个重要信息：
- glibc 上 pthread_join 等待子线程完成，并不是通过传统的 wait/waitpid 实现的，而是由 pthread 自己再维护了一个 futex （在这里作为「线程执行完毕」的条件变量 condition variable），通过等待这个 futex 实现。
- 通过 `__nptl_free_tcb(pd)` 可以知道，所谓的 "TCB" 这个概念实际上就是 pthread 结构体本身（pthread_t 是指向其的指针），并且是存储在用户态的，由 glibc/nptl 管理，而不是在内核态管理。
* pthread_join 只负责释放用户态 pthread 结构体（pd），而和释放线程在内核中占用的资源没有关系。

综合「pthread_join 不负责回收（reap）内核态线程」以及「观察到子线程在执行完成后，在主线程什么都没有做的情况下自己消失了」这两个信息，进一步猜测子线程是退出后被内核自动 reap 掉了。

但是按照正常进程来说，除非是父进程设置了 `signal(SIGCHLD, SIG_IGN);`，否则操作系统是不会自动 reap 掉子进程的，假设内核不区分进程和线程，对线程而言应该也是这个行为（需要等待父进程 reap，否则就处于 ZOMBIE 状态）才对。

由此猜测有可能是两种可能性中的一种：
1. 内核可能对线程 task 有一定的特殊照顾/特殊处理，使得线程的 task 会在退出时自动 reap，而进程则等待父进程回收。
2. 也有一种可能性是 pthread 自己在子线程执行末尾做了特殊处理，让操作系统 reap 掉自己（真的可能做到吗？）

后面的内容和探究都是围绕尝试检验这两个猜想展开的。

## 线程创建 pthread_create()

由于已知线程 task 不是由 pthread_join 回收的，必然是内核或者 pthread 在什么其他地方进行了回收，故追踪整个线程 pthread 从创建开始的生命周期：

<https://codebrowser.dev/glibc/glibc/nptl/pthread_create.c.html>
```c
// pthread_create.c:619
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
		      void *(*start_routine) (void *), void *arg)
{
  void *stackaddr = NULL;
  size_t stacksize = 0;

  // ......

  // 分配 TCB （pthread 结构体）和线程栈空间
  struct pthread *pd = NULL;
  int err = allocate_stack (iattr, &pd, &stackaddr, &stacksize);
  int retval = 0;

  // ......
  
  // 初始化 TCB
  pd->start_routine = start_routine; // 子线程入口
  pd->arg = arg; // 子线程入口参数
  pd->c11 = c11;

  // ......

  /* Setup tcbhead.  */
  tls_setup_tcbhead (pd);

  /* Pass the descriptor to the caller.  */
  *newthread = (pthread_t) pd;

  // ......

  // .......some more signal related stuff

  /* Start the thread.  */
  if (__glibc_unlikely (report_thread_creation (pd))) // 如果需要报告 TD_CREATE 事件
    {
      // ......在我们的例子中不会走到这里，忽略
      // event 是 debug 时用的，例如用来通知 gdb 线程已经创建
    }
  else
    // ！！创建内核态线程 task！！
    retval = create_thread (pd, iattr, &stopped_start, stackaddr,
			    stacksize, &thread_ran);

  // ......

  if (__glibc_unlikely (retval != 0))
    {
      // ......错误处理，忽略
    }
  else
    {
	  // 放开 pd 上的锁，让子线程自由运行
      if (stopped_start) lll_unlock (pd->lock, LLL_PRIVATE);

      // ......
    }

 out:
  if (destroy_default_attr)
    __pthread_attr_destroy (&default_attr.external);

  return retval;
}

```

pthread_create 做的事情并不复杂：
1. 为子线程分配栈空间和 TCB（pd）
2. 准备/配置好 TCB 各项参数，包括子线程入口和入口参数
3. 调用 `create_thread()` 启动子线程 task

## 线程 task 创建 create_thread()

这个函数在 `pthread_create()` 中负责为子线程创建实际的内核态 task，通过调用 clone 实现（`__clone_internal()` 是 clone/clone2/clone3 的简单封装）。

```c
// pthread_create.c:231

static int create_thread (struct pthread *pd, const struct pthread_attr *attr,
			  bool *stopped_start, void *stackaddr,
			  size_t stacksize, bool *thread_ran)
{
  // ......
  
  /* We rely heavily on various flags the CLONE function understands:

     CLONE_VM, CLONE_FS, CLONE_FILES
	These flags select semantics with shared address space and
	file descriptors according to what POSIX requires.

     ...... 篇幅原因缩略，查看原始文件或 `man clone` 查询每个 flag 作用

     The termination signal is chosen to be zero which means no signal
     is sent.  */
  const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD // ！！注意到这个 CLONE_THREAD flag
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);

  TLS_DEFINE_INIT_TP (tp, pd);

  struct clone_args args =
    {
      .flags = clone_flags, // clone flags
      .pidfd = (uintptr_t) &pd->tid,
      .parent_tid = (uintptr_t) &pd->tid, // CLONE_PARENT_SETTID
      .child_tid = (uintptr_t) &pd->tid,
      .stack = (uintptr_t) stackaddr,
      .stack_size = stacksize,
      .tls = (uintptr_t) tp,              // CLONE_SETTLS
    };
  // clone，子线程从 start_thread() 开始执行
  int ret = __clone_internal (&args, &start_thread, pd);
  if (__glibc_unlikely (ret == -1))
    return errno;

  // ......

  return 0;
}

```

这里注意到，调用 clone 的时候，传递给内核的 flags 中含有 CLONE_THREAD 这个 flag。

这个 flag 意味着用户态显式地告知了内核，克隆出来的 task 应该被作为一个线程看待。先不管这个 flag 的具体影响是什么，传递这个 flag 这件事情本身足以说明，内核实际上对普通进程 task 和线程 task 还是有专门的区分的，并不是除了资源共享程度不同以外其他都完全一模一样。

我们前面针对子线程的 task 会被自动 reap 掉这件事，做出了两种猜想：可能是内核特殊处理了线程 task，也可能是 pthread 自己在子线程末尾回收了子线程。

CLONE_THREAD 这个 flag 的存在加大了第一种猜想正确的可能性，不过并不完全排除第二种猜想，要排除第二种猜想，需要看子线程的用户例程执行完毕后，在 pthread 中都做了什么，有没有回收掉子线程 task。

## 子线程 task 执行入口 start_thread()

`create_thread()` 创建的子线程的执行入口固定为 `start_thread()`，这个函数再从 `pd->start_routine` 和 `pd->args` 获得用户函数的地址和参数，并跳转到用户函数开始执行。

```c
/* Local function to start thread and handle cleanup.  */
static int _Noreturn
start_thread (void *arg)
{
  struct pthread *pd = arg;

  // ......

  // ......thread local storage and stuff

  // ......unwinders and stuff, for cancellation and/or exception handling

  // ......

  if (__glibc_likely (! not_first_call))
    {
      /* Store the new cleanup handler info.  */
      THREAD_SETMEM (pd, cleanup_jmp_buf, &unwind_buf);

      __libc_signal_restore_set (&pd->sigmask);

      LIBC_PROBE (pthread_start, 3, (pthread_t) pd, pd->start_routine, pd->arg);

      /* Run the code the user provided.  */
      void *ret;
      if (pd->c11)
	{
	  // ......c11 标准下多了一个类型转换问题，我们不使用 c11 标准，不走到这里，略
	}
      else
	ret = pd->start_routine (pd->arg); // 使用用户提供的参数，调用用户函数，得到返回值 ret
      THREAD_SETMEM (pd, result, ret); // 用户函数返回值 ret 存入到 pd->result
    }

  // ======= 到这里，用户函数已经执行完毕，子线程任务完成，进入结束阶段 ======

  // thread local storage、thread local data 析构
#ifndef SHARED
  if (&__call_tls_dtors != NULL)
#endif
    __call_tls_dtors ();

  /* Run the destructor for the thread-local data.  */
  __nptl_deallocate_tsd ();

  /* Clean up any state libc stored in thread-local variables.  */
  __libc_thread_freeres ();

  /* Report the death of the thread if this is wanted.  */
  if (__glibc_unlikely (pd->report_events))
    {
    // ......在需要时，报告 TD_DEATH 事件，略
    // event 是 debug 时用的，例如用来通知 gdb 线程已经退出
    }

  // ......

  if (__glibc_unlikely (atomic_decrement_and_test (&__nptl_nthreads)))
    /* This was the last thread.  */
    exit (0); // 如果这个线程是最后一个线程，退出整个线程组（thread group）

  // ......signal and mutex and stuff

  // ......如果栈空间是 pthread 分配的（而不是用户创建线程时提供的），则回收栈空间

  // ......

  // 如果线程被 pthread_detach 过，则顺便回收 TCB（注意 TCB 不是 task_struct，是用户态 pthread 结构体）
  // 默认行为是不在这里回收 TCB，我们的例子中也是默认情况，即不会在这里执行 `__nptl_free_tcb(pd)`，而是需要等到 pthread_join() 时才会回收 TCB
  // 这是由于 pd 结构体中含有用户态返回值 pd->result 可能会被父线程需要；pthread_detach 则意味着父线程不关心子线程的执行结果。
  if (IS_DETACHED (pd))
    /* Free the TCB.  */
    __nptl_free_tcb (pd);

out:
  // 子线程完全结束，最后一步：调用 sys_exit 系统调用，结束【子线程】（不是整个进程）
  while (1)
    INTERNAL_SYSCALL_CALL (exit, 0);

  /* NOTREACHED */
}
```

注意到最后一步 `sys_exit` 和常见的 `exit()/_exit()` 不同，前者是系统调用，后者是由 libc 提供的用户态包装方法。

两者的作用效果也不一样：
- `sys_exit` 是退出当前【调度单元】，即 task，在这里是指当前【线程】
- 而 `exit()/_exit()` 实际上包装的是 `sys_exit_group` 系统调用，代表退出整个【线程组】，即整个进程的所有线程。

> 命名的混乱是历史原因，由于一开始 Linux 只支持多进程，最初 `exit()/_exit()` 也的确封装的是 sys_exit。
> 
> 而后来加入多线程后，Linux 在内核态内引入了一个新概念：thread group。原来的进程变成了 task，`task_struct->pid` 变成了线程 id（`gettid()` 返回 `task_struct->pid`），而现在常说的进程 id，则是新的`task_struct->tgid`（thread group id，`getpid()` 返回 `task_struct->tgid` ）。同时，libc 的 `exit()/_exit()` 也被改为调用新的 sys_exit_group，即结束整个线程组。原来的 sys_exit 的作用不变，依然是结束一个调度单元 task，只是「调度单元」的概念改变了而已。
> 
> 这个比较 hack 的方式，使得 linux 以较小的改动，实现了对线程的支持，坏处就是导致用户态的 “pid” 的概念和内核态中的 “pid” 的含义不一致，不留意的话容易混淆。

可以看到，子线程的入口函数 `start_thread()`，在执行完用户函数后，销毁了栈和 thread local storage，然后执行了 sys_exit 结束子线程 task。

这个过程和多进程模型中一个子进程调用 `exit()` 退出线程是类似的，并不保证一定清理掉 task_struct，而是理论上有可能使 task 进入 ZOMBIE 状态。而且这里可以看到，pthread 也没有让子进程做（诸如自己 wait 自己？）之类的魔法来显式清理掉子进程的 task。

这说明我们之前的猜想2是不正确的，子线程的内核 task_struct 并不是 pthread 在子线程退出后进行特殊处理 reap 回收的。只能是 sys_exit 系统调用中，退出子线程 task 的时候，内核自己决定要直接 reap 掉这个 task。

> 实际上猜想2本身也不可能，一个 task 不可能自己回收自己的资源，因为只有已经结束的 task 才能被回收，但是已经结束的 task 就无法执行任何代码了，也就没法回收自己。

## 退出子线程 sys_exit 系统调用

前面通过排除，将我们子线程的 task 被 reap 的准确位置定位到了 sys_exit 中了。我们从用户态的 glibc 以及 pthread，继续进入到内核态代码的范围中。

前一小节结尾提到，我们发现子线程的 task 在用户态是正常 sys_exit 退出的，但是 sys_exit 后 pid  以及 task_struct 被马上回收掉，而不是像普通进程一样进入僵尸状态，这里看到内核 `do_exit()` 方法（sys_exit 系统调用的内核态处理函数）：

<https://elixir.bootlin.com/linux/v5.15/source/kernel/exit.c>
```c
// kernel/exit.c:727
// 不用仔细看这个函数的每一步，这里全放出来只是为了体现步骤有多么多而已
void __noreturn do_exit(long code)
{
	struct task_struct *tsk = current;
	int group_dead;

	// ......

	profile_task_exit(tsk);
	kcov_task_exit(tsk);

	ptrace_event(PTRACE_EVENT_EXIT, code);

	validate_creds_for_do_exit(tsk);

	// ......
	
	io_uring_files_cancel();
	exit_signals(tsk);  /* sets PF_EXITING */

	/* sync mm's RSS info before statistics gathering */
	if (tsk->mm)
		sync_mm_rss(tsk->mm);
	acct_update_integrals(tsk);
	group_dead = atomic_dec_and_test(&tsk->signal->live);
	if (group_dead) {
		/*
		 * If the last thread of global init has exited, panic
		 * immediately to get a useable coredump.
		 */
		if (unlikely(is_global_init(tsk)))
			panic("Attempted to kill init! exitcode=0x%08x\n",
				tsk->signal->group_exit_code ?: (int)code);

#ifdef CONFIG_POSIX_TIMERS
		hrtimer_cancel(&tsk->signal->real_timer);
		exit_itimers(tsk->signal);
#endif
		if (tsk->mm)
			setmax_mm_hiwater_rss(&tsk->signal->maxrss, tsk->mm);
	}
	acct_collect(code, group_dead);
	if (group_dead)
		tty_audit_exit();
	audit_free(tsk);

	tsk->exit_code = code;
	taskstats_exit(tsk, group_dead);

	exit_mm();

	if (group_dead)
		acct_process();
	trace_sched_process_exit(tsk);

	exit_sem(tsk);
	exit_shm(tsk);
	exit_files(tsk);
	exit_fs(tsk);
	if (group_dead)
		disassociate_ctty(1);
	exit_task_namespaces(tsk);
	exit_task_work(tsk);
	exit_thread(tsk);

	perf_event_exit_task(tsk);

	sched_autogroup_exit_task(tsk);
	cgroup_exit(tsk);

	flush_ptrace_hw_breakpoint(tsk);

	exit_tasks_rcu_start();
	exit_notify(tsk, group_dead);
	proc_exit_connector(tsk);
	mpol_put_task_policy(tsk);
#ifdef CONFIG_FUTEX
	if (unlikely(current->pi_state_cache))
		kfree(current->pi_state_cache);
#endif
	/*
	 * Make sure we are holding no locks:
	 */
	debug_check_no_locks_held();

	if (tsk->io_context)
		exit_io_context(tsk);

	if (tsk->splice_pipe)
		free_pipe_info(tsk->splice_pipe);

	if (tsk->task_frag.page)
		put_page(tsk->task_frag.page);

	validate_creds_for_do_exit(tsk);

	check_stack_usage();
	preempt_disable();
	if (tsk->nr_dirtied)
		__this_cpu_add(dirty_throttle_leaks, tsk->nr_dirtied);
	exit_rcu();
	exit_tasks_rcu_finish();

	lockdep_free_task(tsk);
	do_task_dead();
}
EXPORT_SYMBOL_GPL(do_exit);
```

可以看到一个 task 退出时需要清理/释放的资源种类非常之多，主流程`do_exit()`里的子流程函数调用就有好几十个了。

我们想要从中找到这个逻辑：exit 的时候，内核根据什么决定是否直接 reap 掉 task。

利用已知知识帮助快速定位到想要找的逻辑的代码：已知对于一般进程来说，如果父进程设置了忽略 SIGCHLD 信号（`signal(SIGCHLD, SIG_IGN)`），则子进程 exit 的时候会【reap 掉 task】，否则子进程会【进入 ZOMBIE 状态】。这实际上正是我们要找的「exit 决定是否直接 reap 掉 task」的决策过程的一部分。猜测对于线程 task 是否自动 reap 的决策逻辑也是在相同的位置或附近。

故以【进入 ZOMBIE 状态】为线索反查，直接搜索 ` = EXIT_ZOMBIE` 尝试找所有将 task 状态设置为 ZOMBIE 的地方，快速定位到 `exit_notify()` 中：

```c
/*
 * Send signals to all our closest relatives so that they know
 * to properly mourn us..
 */
static void exit_notify(struct task_struct *tsk, int group_dead)
{
	bool autoreap; // 是否自动 reap 掉 task

	// ......

	tsk->exit_state = EXIT_ZOMBIE; // 默认将 task 置入 EXIT_ZOMBIE 状态
	if (unlikely(tsk->ptrace)) { // 如果启用了 ptrace（一般用于 debug）
		int sig = // ......
		autoreap = do_notify_parent(tsk, sig);
	} else if (thread_group_leader(tsk)) { // 如果是线程组组长，即主线程
	    // 并且整个线程组（进程）中所有线程都已经退出
	    // 则发送 SIGCHLD 给 parent，如果父进程 SIG_IGN 掉了 SIGCHLD，则自动 reap
		autoreap = thread_group_empty(tsk) &&
			do_notify_parent(tsk, tsk->exit_signal);
	} else { // 其他任何情况，即：不是线程组组长（例如子线程）
		autoreap = true; // 则均自动 reap
	}

	if (autoreap) { // 自动 reap 的进程直接进入 EXIT_DEAD 状态
		tsk->exit_state = EXIT_DEAD;
		list_add(&tsk->ptrace_entry, &dead);
	}

	// ......
}
```

这里证实了猜想1：当一个 task 不是一个线程组的组长的时候，内核会在 exit 的时候直接 reap 掉子线程的 task。所以在子线程执行完毕但是未 join 之间，ps 会看不到子线程，因为子线程 task 已经被内核回收了。

也就是说，**只有一个线程组（也就是进程）的主线程可以进入僵尸状态，所有的子线程都不可能会有僵尸状态。子线程的 task 在用户程序代码执行完毕后，就马上退出并被回收了。** 

而子线程的执行结果（返回值等）则保存在用户态 pthread 结构体中，供后续可能的 JOIN 使用。

# 结论

对于 Linux 平台上的 pthread 线程，在子线程比父线程先退出且没被 JOIN 的情况下，不会产生和传统意义上的僵尸进程类似的“僵尸线程”（即 ps 不会看到有 defunct 的线程 task，子线程 task 会在 exit 时被内核直接回收掉，不等父进程 JOIN）。

但是并不意味着没有被 pthread_join 的线程完全不会占用资源。

没被回收的线程虽然不会占用内核的 task 资源，但是会在用户态留下 pthread 结构体（TCB）以及线程的栈（因为 pthread 结构体和线程栈是一起分配一起释放的），如果未被 JOIN 的线程累积过多，仍然可能会导致用户态资源耗尽而导致**该进程**无法创建新的线程。

与僵尸进程不同的是，“僵尸线程”堆积的影响只限制在一个进程之内，理论上不会导致系统上其他进程创建失败（因为不占用 task_struct 和 pid/tid）。

> 注意到该结论只适用于 Linux，因为 Linux 实现线程的方式为内核轻改动，大多数线程相关的功能实现都在用户态中实现（glibc）。不排除其他 POSIX 系统可能内核级原生支持 pthread 线程。

`pthread_detach()` 过的线程，则 pthread 会在线程执行完成后自动释放 pthread 结构体以及栈，所以对不关心执行结果的线程，应当使用 `pthread_detach()` 进行脱离。若不进行脱离，则必须确保所有线程在合适的时间都能被 `pthread_join()` 回收。

> Reference:
> <https://manpages.debian.org/bullseye/manpages-dev/clone.2.en.html#CLONE_THREAD>
> <https://codebrowser.dev/glibc/glibc/nptl/>
> <https://elixir.bootlin.com/linux/v5.15/source/kernel/exit.c>
