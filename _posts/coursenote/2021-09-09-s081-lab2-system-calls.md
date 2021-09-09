---
title: "[mit6.s081] 笔记 Lab2: System calls | 系统调用"
date: 2021-09-09 19:00:00 +0800
categories: [Course Notes, MIT6.S081]
tags: [operating system]
---
> 这是我自学 MIT6.S081 操作系统课程的 lab 代码笔记第二篇：System calls。此 lab 大致耗时：4小时。  
> 
> 课程地址：[https://pdos.csail.mit.edu/6.S081/2020/schedule.html](https://pdos.csail.mit.edu/6.S081/2020/schedule.html)  
> Lab 地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/syscall.html](https://pdos.csail.mit.edu/6.S081/2020/labs/syscall.html)  
> 我的代码地址：[https://github.com/Miigon/my-xv6-labs-2020/tree/syscall](https://github.com/Miigon/my-xv6-labs-2020/tree/syscall)  
> Commits: [https://github.com/Miigon/my-xv6-labs-2020/commits/syscall](https://github.com/Miigon/my-xv6-labs-2020/commits/syscall)  
> 
> 本文中代码注释是编写博客的时候加入的，原仓库中的代码可能缺乏注释或代码不完全相同。  

# Lab 2: System calls

In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. You will add more system calls in later labs.

对 xv6 添加一些新的系统调用，帮助加深对 xv6 内核的理解。

# System call tracing (moderate)
准备环境，编译编译器、QEMU，克隆仓库，略过。
> In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new trace system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls trace(1 << SYS_fork), where SYS_fork is a syscall number from kernel/syscall.h. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The trace system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

添加一个系统调用 trace 的功能，为每个进程设定一个位 mask，用 mask 中设定的位来指定要为哪些系统调用输出调试信息。

## 如何创建新系统调用

1. 首先在内核中合适的位置（取决于要实现的功能属于什么模块，理论上随便放都可以，只是主要起归类作用），实现我们的内核调用（在这里是 trace 调用）：  
	```c
	// kernel/sysproc.c
	// 这里着重理解如何添加系统调用，对于这个调用的具体代码细节在后面的部分分析
	uint64
	sys_trace(void)
	{
	int mask;

	if(argint(0, &mask) < 0)
		return -1;
	
	myproc()->syscall_trace = mask;
	return 0;
	}
	```
	这里因为我们的系统调用会对进程进行操作，所以放在 sysproc.c 较为合适。
2. 在 syscall.h 中加入新 system call 的序号：
	```c
	// kernel/syscall.h
	// System call numbers
	#define SYS_fork    1
	#define SYS_exit    2
	#define SYS_wait    3
	#define SYS_pipe    4
	#define SYS_read    5
	#define SYS_kill    6
	#define SYS_exec    7
	#define SYS_fstat   8
	#define SYS_chdir   9
	#define SYS_dup    10
	#define SYS_getpid 11
	#define SYS_sbrk   12
	#define SYS_sleep  13
	#define SYS_uptime 14
	#define SYS_open   15
	#define SYS_write  16
	#define SYS_mknod  17
	#define SYS_unlink 18
	#define SYS_link   19
	#define SYS_mkdir  20
	#define SYS_close  21
	#define SYS_trace  22 // here!!!!!
	```
3. 用 extern 全局声明新的内核调用函数，并且在 syscalls 映射表中，加入从前面定义的编号到系统调用函数指针的映射
	```c
	// kernel/syscall.c 
	extern uint64 sys_chdir(void);
	extern uint64 sys_close(void);
	extern uint64 sys_dup(void);
	extern uint64 sys_exec(void);
	extern uint64 sys_exit(void);
	extern uint64 sys_fork(void);
	extern uint64 sys_fstat(void);
	extern uint64 sys_getpid(void);
	extern uint64 sys_kill(void);
	extern uint64 sys_link(void);
	extern uint64 sys_mkdir(void);
	extern uint64 sys_mknod(void);
	extern uint64 sys_open(void);
	extern uint64 sys_pipe(void);
	extern uint64 sys_read(void);
	extern uint64 sys_sbrk(void);
	extern uint64 sys_sleep(void);
	extern uint64 sys_unlink(void);
	extern uint64 sys_wait(void);
	extern uint64 sys_write(void);
	extern uint64 sys_uptime(void);
	extern uint64 sys_trace(void);   // HERE

	static uint64 (*syscalls[])(void) = {
	[SYS_fork]    sys_fork,
	[SYS_exit]    sys_exit,
	[SYS_wait]    sys_wait,
	[SYS_pipe]    sys_pipe,
	[SYS_read]    sys_read,
	[SYS_kill]    sys_kill,
	[SYS_exec]    sys_exec,
	[SYS_fstat]   sys_fstat,
	[SYS_chdir]   sys_chdir,
	[SYS_dup]     sys_dup,
	[SYS_getpid]  sys_getpid,
	[SYS_sbrk]    sys_sbrk,
	[SYS_sleep]   sys_sleep,
	[SYS_uptime]  sys_uptime,
	[SYS_open]    sys_open,
	[SYS_write]   sys_write,
	[SYS_mknod]   sys_mknod,
	[SYS_unlink]  sys_unlink,
	[SYS_link]    sys_link,
	[SYS_mkdir]   sys_mkdir,
	[SYS_close]   sys_close,
	[SYS_trace]   sys_trace,  // AND HERE
	};
	```
	这里 `[SYS_trace] sys_trace` 是 C 语言数组的一个语法，表示以方括号内的值作为元素下标。比如 `int arr[] = {[3] 2333, [6] 6666}` 代表 arr 的下标 3 的元素为 2333，下标 6 的元素为 6666，其他元素填充 0 的数组。（该语法在 C++ 中已不可用）
4. 在 usys.pl 中，加入用户态到内核态的跳板函数。
	```perl
	# user/usys.pl

	entry("fork");
	entry("exit");
	entry("wait");
	entry("pipe");
	entry("read");
	entry("write");
	entry("close");
	entry("kill");
	entry("exec");
	entry("open");
	entry("mknod");
	entry("unlink");
	entry("fstat");
	entry("link");
	entry("mkdir");
	entry("chdir");
	entry("dup");
	entry("getpid");
	entry("sbrk");
	entry("sleep");
	entry("uptime");
	entry("trace");  # HERE
	```
	这个脚本在运行后会生成 usys.S 汇编文件，里面定义了每个 system call 的用户态跳板函数：
	```asm
	trace:		# 定义用户态跳板函数
	li a7, SYS_trace	# 将系统调用 id 存入 a7 寄存器
	ecall				# ecall，调用 system call ，跳到内核态的统一系统调用处理函数 syscall()  (syscall.c)
	ret
	```
5. 在用户态的头文件加入定义，使得用户态程序可以找到这个跳板入口函数。
	```c
	// user/user.h
	// system calls
	int fork(void);
	int exit(int) __attribute__((noreturn));
	int wait(int*);
	int pipe(int*);
	int write(int, const void*, int);
	int read(int, void*, int);
	int close(int);
	int kill(int);
	int exec(char*, char**);
	int open(const char*, int);
	int mknod(const char*, short, short);
	int unlink(const char*);
	int fstat(int fd, struct stat*);
	int link(const char*, const char*);
	int mkdir(const char*);
	int chdir(const char*);
	int dup(int);
	int getpid(void);
	char* sbrk(int);
	int sleep(int);
	int uptime(void);
	int trace(int);		// HERE
	```

## 系统调用全流程

```
user/user.h:		用户态程序调用跳板函数 trace()
user/usys.S:		跳板函数 trace() 使用 CPU 提供的 ecall 指令，调用到内核态
kernel/syscall.c	到达内核态统一系统调用处理函数 syscall()，所有系统调用都会跳到这里来处理。
kernel/syscall.c	syscall() 根据跳板传进来的系统调用编号，查询 syscalls[] 表，找到对应的内核函数并调用。
kernel/sysproc.c	到达 sys_trace() 函数，执行具体内核操作
```
这么繁琐的调用流程的主要目的是实现用户态和内核态的良好隔离。

并且由于内核与用户进程的页表不同，寄存器也不互通，所以参数无法直接通过 C 语言参数的形式传过来，而是需要使用 argaddr、argint、argstr 等系列函数，从进程的 trapframe 中读取用户进程寄存器中的参数。

同时由于页表不同，指针也不能直接互通访问（也就是内核不能直接对用户态传进来的指针进行解引用），而是需要使用 copyin、copyout 方法结合进程的页表，才能顺利找到用户态指针（逻辑地址）对应的物理内存地址。（在本 lab 第二个实验会用到）

```
struct proc *p = myproc(); // 获取调用该 system call 的进程的 proc 结构
copyout(p->pagetable, addr, (char *)&data, sizeof(data)); // 将内核态的 data 变量（常为struct），结合进程的页表，写到进程内存空间内的 addr 地址处。
```

## 该 lab 代码

首先在 proc.h 中修改 proc 结构的定义，添加 syscall_trace field，用 mask 的方式记录要 trace 的 system call。
```c
// kernel/proc.h
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  uint64 syscall_trace;        // Mask for syscall tracing (新添加的用于标识追踪哪些 system call 的 mask)
};
```

在 proc.c 中，创建新进程的时候，为新添加的 syscall_trace 附上默认值 0（否则初始状态下可能会有垃圾数据）。

```c
// kernel/proc.c
static struct proc*
allocproc(void)
{
  ......

  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  p->syscall_trace = 0; // (newly added) 为 syscall_trace 设置一个 0 的默认值

  return p;
}
```

在 sysproc.c 中，实现 system call 的具体代码，也就是设置当前进程的 syscall_trace mask：

```c
// kernel/sysproc.c
uint64
sys_trace(void)
{
  int mask;

  if(argint(0, &mask) < 0) // 通过读取进程的 trapframe，获得 mask 参数
    return -1;
  
  myproc()->syscall_trace = mask; // 设置调用进程的 syscall_trace mask
  return 0;
}
```

修改 fork 函数，使得子进程可以继承父进程的 syscall_trace mask：

```c
// kernel/proc.c
int
fork(void)
{
  ......

  safestrcpy(np->name, p->name, sizeof(p->name));

  np->syscall_trace = p->syscall_trace; // HERE!!! 子进程继承父进程的 syscall_trace

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}
```

根据上方提到的系统调用的全流程，可以知道，所有的系统调用到达内核态后，都会进入到 syscall() 这个函数进行处理，所以要跟踪所有的内核函数，只需要在 syscall() 函数里埋点就行了。

```c
// kernel/syscall.c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) { // 如果系统调用编号有效
    p->trapframe->a0 = syscalls[num](); // 通过系统调用编号，获取系统调用处理函数的指针，调用并将返回值存到用户进程的 a0 寄存器中
	// 如果当前进程设置了对该编号系统调用的 trace，则打出 pid、系统调用名称和返回值。
    if((p->syscall_trace >> num) & 1) {
      printf("%d: syscall %s -> %d\n",p->pid, syscall_names[num], p->trapframe->a0); // syscall_names[num]: 从 syscall 编号到 syscall 名的映射表
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

上面打出日志的过程还需要知道系统调用的名称字符串，在这里定义一个字符串数组进行映射：
```c
// kernel/syscall.c
const char *syscall_names[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace",
};
```

编译执行：
```
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
$
$ trace 2147483647 grep hello README
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
4: syscall close -> 0
$
```

成功追踪并打印出相应的系统调用。

# Sysinfo (moderate)

> In this assignment you will add a system call, sysinfo, that collects information about the running system. The system call takes one argument: a pointer to a struct sysinfo (see kernel/sysinfo.h). The kernel should fill out the fields of this struct: the freemem field should be set to the number of bytes of free memory, and the nproc field should be set to the number of processes whose state is not UNUSED. We provide a test program sysinfotest; you pass this assignment if it prints "sysinfotest: OK".

添加一个系统调用，返回空闲的内存、以及已创建的进程数量。大多数步骤和上个实验是一样的，所以不再描述。唯一不同就是需要把结构体从内核内存拷贝到用户进程内存中。其他的难点可能就是在如何获取空闲内存和如何获取已创建进程上面了，因为涉及到了一些后面的知识。

## 获取空闲内存

在内核的头文件中声明计算空闲内存的函数，因为是内存相关的，所以放在 kalloc、kfree 等函数的的声明之后。
```c
// kernel/defs.h
void*           kalloc(void);
void            kfree(void *);
void            kinit(void);
uint64 			count_free_mem(void); // here
```

在 kalloc.c 中添加计算空闲内存的函数：
```c
// kernel/kalloc.c
uint64
count_free_mem(void) // added for counting free memory in bytes (lab2)
{
  acquire(&kmem.lock); // 必须先锁内存管理结构，防止竞态条件出现
  
  // 统计空闲页数，乘上页大小 PGSIZE 就是空闲的内存字节数
  uint64 mem_bytes = 0;
  struct run *r = kmem.freelist;
  while(r){
    mem_bytes += PGSIZE;
    r = r->next;
  }

  release(&kmem.lock);

  return mem_bytes;
}
```

xv6 中，空闲内存页的记录方式是，将空虚内存页**本身**直接用作链表节点，形成一个空闲页链表，每次需要分配，就把链表根部对应的页分配出去。每次需要回收，就把这个页作为新的根节点，把原来的 freelist 链表接到后面。注意这里是**直接使用空闲页本身**作为链表节点，所以不需要使用额外空间来存储空闲页链表，在 kalloc() 里也可以看到，分配内存的最后一个阶段，是直接将 freelist 的根节点地址（物理地址）返回出去了：
```c
// kernel/kalloc.c
// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist; // 获得空闲页链表的根节点
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r; // 把空闲页链表的根节点返回出去，作为内存页使用（长度是 4096）
}
```

**常见的记录空闲页的方法有：空闲表法、空闲链表法、位示图法（位图法）、成组链接法**。这里 xv6 采用的是空闲链表法。

## 获取运行的进程数

同样在内核的头文件中添加函数声明：
```c
// kernel/defs.h
......
void            sleep(void*, struct spinlock*);
void            userinit(void);
int             wait(uint64);
void            wakeup(void*);
void            yield(void);
int             either_copyout(int user_dst, uint64 dst, void *src, uint64 len);
int             either_copyin(void *dst, int user_src, uint64 src, uint64 len);
void            procdump(void);
uint64			count_process(void); // here
```

在 proc.c 中实现该函数：
```c
uint64
count_process(void) { // added function for counting used process slots (lab2)
  uint64 cnt = 0;
  for(struct proc *p = proc; p < &proc[NPROC]; p++) {
    // acquire(&p->lock);
    // 不需要锁进程 proc 结构，因为我们只需要读取进程列表，不需要写
    if(p->state != UNUSED) { // 不是 UNUSED 的进程位，就是已经分配的
        cnt++;
    }
  }
  return cnt;
}
```

## 实现 sysinfo 系统调用

添加系统调用的流程与实验 1 类似，不再赘述。

这是具体系统信息函数的实现，其中调用了前面实现的 count_free_mem() 和 count_process()：

```c
uint64
sys_sysinfo(void)
{
  // 从用户态读入一个指针，作为存放 sysinfo 结构的缓冲区
  uint64 addr;
  if(argaddr(0, &addr) < 0)
    return -1;
  
  struct sysinfo sinfo;
  sinfo.freemem = count_free_mem(); // kalloc.c
  sinfo.nproc = count_process(); // proc.c
  
  // 使用 copyout，结合当前进程的页表，获得进程传进来的指针（逻辑地址）对应的物理地址
  // 然后将 &sinfo 中的数据复制到该指针所指位置，供用户进程使用。
  if(copyout(myproc()->pagetable, addr, (char *)&sinfo, sizeof(sinfo)) < 0)
    return -1;
  return 0;
}
```

在 user.h 提供用户态入口：

```
// user.h
char* sbrk(int);
int sleep(int);
int uptime(void);
int trace(int);
struct sysinfo; // 这里要声明一下 sysinfo 结构，供用户态使用。
int sysinfo(struct sysinfo *);
```

编译运行：
```
$ sysinfotest
sysinfotest: start
sysinfotest: OK
```

# Optional challenges
Print the system call arguments for traced system calls (easy). （跳过）  
Compute the load average and export it through sysinfo(moderate). （跳过）