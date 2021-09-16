---
title: "[mit6.s081] 笔记 Lab3: Page tables | 页表"
date: 2021-09-16 19:00:00 +0800
categories: [Course Notes, MIT6.S081]
tags: [operating system]
---
> 这是我自学 MIT6.S081 操作系统课程的 lab 代码笔记第三篇：Page tables。此 lab 大致耗时：19小时。  
> 
> 课程地址：[https://pdos.csail.mit.edu/6.S081/2020/schedule.html](https://pdos.csail.mit.edu/6.S081/2020/schedule.html)  
> Lab 地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/pgtbl.html](https://pdos.csail.mit.edu/6.S081/2020/labs/pgtbl.html)  
> 我的代码地址：[https://github.com/Miigon/my-xv6-labs-2020/tree/pgtbl](https://github.com/Miigon/my-xv6-labs-2020/tree/pgtbl)  
> Commits: [https://github.com/Miigon/my-xv6-labs-2020/commits/pgtbl](https://github.com/Miigon/my-xv6-labs-2020/commits/pgtbl)  
> 
> 本文中代码注释是编写博客的时候加入的，原仓库中的代码可能缺乏注释或代码不完全相同。  

# Lab 3: Page tables

In this lab you will explore page tables and modify them to simplify the functions that copy data from user space to kernel space.

对 xv6 添加一些新的系统调用，帮助加深对 xv6 内核的理解。

## Print a page table (easy)

> Define a function called vmprint(). It should take a pagetable_t argument, and print that pagetable in the format described below. Insert if(p->pid==1) vmprint(p->pagetable) in exec.c just before the return argc, to print the first process's page table. You receive full credit for this assignment if you pass the pte printout test of make grade.

添加一个打印页表的内核函数，以如如下格式打印出传进的页表，用于后面两个实验调试用：

```text
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

RISC-V 的逻辑地址寻址是采用三级页表的形式，9 bit 一级索引找到二级页表，9 bit 二级索引找到三级页表，9 bit 三级索引找到内存页，最低 12 bit 为页内偏移（即一个页 4096 bytes）。具体可以参考 [xv6 book](https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf) 的 Figure 3.2。

本函数需要模拟如上的 CPU 查询页表的过程，对三级页表进行遍历，然后按照一定格式输出

```c
// kernel/defs.h
......
int             copyout(pagetable_t, uint64, char *, uint64);
int             copyin(pagetable_t, char *, uint64, uint64);
int             copyinstr(pagetable_t, char *, uint64, uint64);
int             vmprint(pagetable_t pagetable); // 添加函数声明
```

因为需要递归打印页表，而 xv6 已经有一个递归释放页表的函数 freewalk()，将其复制一份，并将释放部分代码改为打印即可：

```c
// kernel/vm.c
int pgtblprint(pagetable_t pagetable, int depth) {
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V) { // 如果页表项有效
      // 按格式打印页表项
      printf("..");
      for(int j=0;j<depth;j++) {
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));

      // 如果该节点不是叶节点，递归打印其子节点。
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // this PTE points to a lower-level page table.
        uint64 child = PTE2PA(pte);
        pgtblprint((pagetable_t)child,depth+1);
      }
    }
  }
  return 0;
}

int vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  return pgtblprint(pagetable, 0);
}
```

```c
// exec.c

int
exec(char *path, char **argv)
{
  // ......

  vmprint(p->pagetable); // 按照实验要求，在 exec 返回之前打印一下页表。
  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;

}
```

grade:
```
$ ./grade-lab-pgtbl pte printout
make: `kernel/kernel' is up to date.
== Test pte printout == pte printout: OK (1.6s) 
```

## A kernel page table per process (hard)

> Your first job is to modify the kernel so that every process uses its own copy of the kernel page table when executing in the kernel. Modify struct proc to maintain a kernel page table for each process, and modify the scheduler to switch kernel page tables when switching processes. For this step, each per-process kernel page table should be identical to the existing global kernel page table. You pass this part of the lab if usertests runs correctly.

xv6 原本的设计是，用户进程在用户态使用各自的用户态页表，但是一旦进入内核态（例如使用了系统调用），则切换到内核页表（通过修改 satp 寄存器，trampoline.S）。然而这个内核页表是全局共享的，也就是全部进程进入内核态都共用同一个内核态页表：
```c
// vm.c
pagetable_t kernel_pagetable; // 全局变量，共享的内核页表
```

本 Lab 目标是让每一个进程进入内核态后，都能有自己的独立**内核页表**，为第三个实验做准备。

### 创建进程内核页表与内核栈

首先在进程的结构体 proc 中，添加一个 kernelpgtbl，用于存储进程专享的内核态页表。
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
  pagetable_t kernelpgtbl;     // Kernel page table （在 proc 中添加该 field）
};
```

接下来暴改 kvminit。内核需要依赖内核页表内一些固定的映射的存在才能正常工作，例如 UART 控制、硬盘界面、中断控制等。而 kvminit 原本只为全局内核页表 kernel_pagetable 添加这些映射。我们抽象出来一个可以为任何我们自己创建的内核页表添加这些映射的函数 kvm_map_pagetable()。

```c
void kvm_map_pagetable(pagetable_t pgtbl) {
  // 将各种内核需要的 direct mapping 添加到页表 pgtbl 中。
  
  // uart registers
  kvmmap(pgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(pgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmap(pgtbl, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap(pgtbl, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(pgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap(pgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(pgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}

pagetable_t
kvminit_newpgtbl()
{
  pagetable_t pgtbl = (pagetable_t) kalloc();
  memset(pgtbl, 0, PGSIZE);

  kvm_map_pagetable(pgtbl);

  return pgtbl;
}

/*
 * create a direct-map page table for the kernel.
 */
void
kvminit()
{
  kernel_pagetable = kvminit_newpgtbl(); // 仍然需要有全局的内核页表，用于内核 boot 过程，以及无进程在运行时使用。
}

// ......

// 将某个逻辑地址映射到某个物理地址（添加第一个参数 pgtbl）
void
kvmmap(pagetable_t pgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(pgtbl, va, sz, pa, perm) != 0)
    panic("kvmmap");
}

// kvmpa 将内核逻辑地址转换为物理地址（添加第一个参数 kernelpgtbl）
uint64
kvmpa(pagetable_t pgtbl, uint64 va)
{
  uint64 off = va % PGSIZE;
  pte_t *pte;
  uint64 pa;

  pte = walk(pgtbl, va, 0);
  if(pte == 0)
    panic("kvmpa");
  if((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa+off;
}

```

现在可以创建进程间相互独立的内核页表了，但是还有一个东西需要处理：内核栈。
原本的 xv6 设计中，所有处于内核态的进程都共享同一个页表，即意味着共享同一个地址空间。由于 xv6 支持多核/多进程调度，同一时间可能会有多个进程处于内核态，所以需要对所有处于内核态的进程创建其独立的内核态内的栈，也就是内核栈，供给其内核态代码执行过程。

xv6 在启动过程中，会在 procinit() 中为所有可能的 64 个进程位都预分配好内核栈 kstack，具体为在高地址空间里，每个进程使用一个页作为 kstack，并且两个不同 kstack 中间隔着一个无映射的 guard page 用于检测栈溢出错误。具体参考 [xv6 book](https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf) 的 Figure 3.3。

在 xv6 原来的设计中，内核页表本来是只有一个的，所有进程共用，所以需要为不同进程创建多个内核栈，并 map 到不同位置（见 `procinit()` 和 `KSTACK` 宏）。而我们的新设计中，每一个进程都会有自己独立的内核页表，并且每个进程也只需要访问自己的内核栈，而不需要能够访问所有 64 个进程的内核栈。所以可以将所有进程的内核栈 map 到其**各自内核页表内的固定位置**（不同页表内的同一逻辑地址，指向不同物理内存）。


```
// initialize the proc table at boot time.
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");

      // 这里删除了为所有进程预分配内核栈的代码，变为创建进程的时候再创建内核栈，见 allocproc()
  }

  kvminithart();
}

```

然后，在创建进程的时候，为进程分配独立的内核页表，以及内核栈

```c
// kernel/proc.c

static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

////// 新加部分 start //////

  // 为新进程创建独立的内核页表，并将内核所需要的各种映射添加到新页表上
  p->kernelpgtbl = kvminit_newpgtbl();
  // printf("kernel_pagetable: %p\n", p->kernelpgtbl);

  // 分配一个物理页，作为新进程的内核栈使用
  char *pa = kalloc();
  if(pa == 0)
    panic("kalloc");
  uint64 va = KSTACK((int)0); // 将内核栈映射到固定的逻辑地址上
  // printf("map krnlstack va: %p to pa: %p\n", va, pa);
  kvmmap(p->kernelpgtbl, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va; // 记录内核栈的逻辑地址，其实已经是固定的了，依然这样记录是为了避免需要修改其他部分 xv6 代码

////// 新加部分 end //////

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

到这里进程独立的内核页表就创建完成了，但是目前只是创建而已，用户进程进入内核态后依然会使用全局共享的内核页表，因此还需要在 scheduler() 中进行相关修改。

### 切换到进程内核页表

在调度器将 CPU 交给进程执行之前，切换到该进程对应的内核页表：

```c
// kernel/proc.c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;

        // 切换到进程独立的内核页表
        w_satp(MAKE_SATP(p->kernelpgtbl));
        sfence_vma(); // 清除快表缓存
        
        // 调度，执行进程
        swtch(&c->context, &p->context);

        // 切换回全局内核页表
        kvminithart();

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}

```

到这里，每个进程执行的时候，就都会在内核态采用自己独立的内核页表了。  

### 释放进程内核页表

最后需要做的事情就是在进程结束后，应该释放进程独享的页表以及内核栈，回收资源，否则会导致内存泄漏。

（如果 usertests 在 reparent2 的时候出现了 `panic: kvmmap`，大概率是因为大量内存泄漏消耗完了内存，导致 kvmmap 分配页表项所需内存失败，这时候应该检查是否正确释放了每一处分配的内存，尤其是页表是否每个页表项都释放干净了，）

```c
// kernel/proc.c
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  
  // 释放进程的内核栈
  void *kstack_pa = (void *)kvmpa(p->kernelpgtbl, p->kstack);
  // printf("trace: free kstack %p\n", kstack_pa);
  kfree(kstack_pa);
  p->kstack = 0;
  
  // 注意：此处不能使用 proc_freepagetable，因为其不仅会释放页表本身，还会把页表内所有的叶节点对应的物理页也释放掉。
  // 这会导致内核运行所需要的关键物理页被释放，从而导致内核崩溃。
  // 这里使用 kfree(p->kernelpgtbl) 也是不足够的，因为这只释放了**一级页表本身**，而不释放二级以及三级页表所占用的空间。
  
  // 递归释放进程独享的页表，释放页表本身所占用的空间，但**不释放页表指向的物理页**
  kvm_free_kernelpgtbl(p->kernelpgtbl);
  p->kernelpgtbl = 0;
  p->state = UNUSED;
}
```

kvm_free_kernelpgtbl() 用于递归释放整个多级页表树，也是从 freewalk() 修改而来。

```c
// kernel/vm.c

// 递归释放一个内核页表中的所有 mapping，但是不释放其指向的物理页
void
kvm_free_kernelpgtbl(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    uint64 child = PTE2PA(pte);
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){ // 如果该页表项指向更低一级的页表
      // 递归释放低一级页表及其页表项
      kvm_free_kernelpgtbl((pagetable_t)child);
      pagetable[i] = 0;
    }
  }
  kfree((void*)pagetable); // 释放当前级别页表所占用空间
}

```

这里释放部分就实现完成了。

注意到我们的修改影响了其他代码： virtio 磁盘驱动 virtio_disk.c 中调用了 kvmpa() 用于将虚拟地址转换为物理地址，这一操作在我们修改后的版本中，需要传入进程的内核页表。对应修改即可。
```c
// virtio_disk.c
#include "proc.h" // 添加头文件引入

// ......

void
virtio_disk_rw(struct buf *b, int write)
{
// ......
disk.desc[idx[0]].addr = (uint64) kvmpa(myproc()->kernelpgtbl, (uint64) &buf0); // 调用 myproc()，获取进程内核页表
// ......
}
```

## Simplify copyin/copyinstr (hard)

> Replace the body of copyin in kernel/vm.c with a call to copyin_new (defined in kernel/vmcopyin.c); do the same for copyinstr and copyinstr_new. Add mappings for user addresses to each process's kernel page table so that copyin_new and copyinstr_new work. You pass this assignment if usertests runs correctly and all the make grade tests pass.

在上一个实验中，已经使得每一个进程都拥有独立的内核态页表了，这个实验的目标是，在进程的内核态页表中维护一个用户态页表映射的副本，这样使得内核态也可以对用户态传进来的指针（逻辑地址）进行解引用。这样做相比原来 copyin 的实现的优势是，原来的 copyin 是通过软件模拟访问页表的过程获取物理地址的，而在内核页表内维护映射副本的话，可以利用 CPU 的硬件寻址功能进行寻址，效率更高并且可以受快表加速。

要实现这样的效果，我们需要在每一处内核对用户页表进行修改的时候，将同样的修改也同步应用在进程的内核页表上，使得两个页表的程序段（0 到 PLIC 段）地址空间的映射同步。

### 准备

首先实现一些工具方法，多数是参考现有方法改造得来：

```c
// kernel/vm.c

// 注：需要在 defs.h 中添加相应的函数声明，这里省略。

// 将 src 页表的一部分页映射关系拷贝到 dst 页表中。
// 只拷贝页表项，不拷贝实际的物理页内存。
// 成功返回0，失败返回 -1
int
kvmcopymappings(pagetable_t src, pagetable_t dst, uint64 start, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  // PGROUNDUP: 对齐页边界，防止 remap
  for(i = PGROUNDUP(start); i < start + sz; i += PGSIZE){
    if((pte = walk(src, i, 0)) == 0)
      panic("kvmcopymappings: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("kvmcopymappings: page not present");
    pa = PTE2PA(*pte);
    // `& ~PTE_U` 表示将该页的权限设置为非用户页
    // 必须设置该权限，RISC-V 中内核是无法直接访问用户页的。
    flags = PTE_FLAGS(*pte) & ~PTE_U;
    if(mappages(dst, i, PGSIZE, pa, flags) != 0){
      goto err;
    }
  }

  return 0;

 err:
  uvmunmap(dst, 0, i / PGSIZE, 0);
  return -1;
}

// 与 uvmdealloc 功能类似，将程序内存从 oldsz 缩减到 newsz。但区别在于不释放实际内存
// 用于内核页表内程序内存映射与用户页表程序内存映射之间的同步
uint64
kvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 0);
  }

  return newsz;
}

```

接下来，为映射程序内存做准备。实验中提示内核启动后，能够用于映射程序内存的地址范围是 [0,PLIC)，我们将把进程程序内存映射到其内核页表的这个范围内，首先要确保这个范围没有和其他映射冲突。

查阅 xv6 book 可以看到，在 PLIC 之前还有一个 CLINT（核心本地中断器）的映射，该映射会与我们要 map 的程序内存冲突。查阅 xv6 book 的 Chapter 5 以及 start.c 可以知道 CLINT 仅在内核启动的时候需要使用到，而用户进程在内核态中的操作并不需要使用到该映射。

![Figure 3.3](/assets/img/mit6s081-lab3-figure-3-3.png)

所以修改 kvm_map_pagetable()，去除 CLINT 的映射，这样进程内核页表就不会有 CLINT 与程序内存映射冲突的问题。但是由于全局内核页表也使用了 kvm_map_pagetable() 进行初始化，并且内核启动的时候需要 CLINT 映射存在，故在 kvminit() 中，另外单独给全局内核页表映射 CLINT。

```c
// kernel/vm.c


void kvm_map_pagetable(pagetable_t pgtbl) {
  
  // uart registers
  kvmmap(pgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(pgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  // kvmmap(pgtbl, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap(pgtbl, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // ......
}

// ......

void
kvminit()
{
  kernel_pagetable = kvminit_newpgtbl();
  // CLINT *is* however required during kernel boot up and
  // we should map it for the global kernel pagetable
  kvmmap(kernel_pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
}

```

同时在 exec 中加入检查，防止程序内存超过 PLIC：
```c
int
exec(char *path, char **argv)
{
  // ......

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    if(sz1 >= PLIC) { // 添加检测，防止程序大小超过 PLIC
      goto bad;
    }
    sz = sz1;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;
  // .......
```

### 同步映射

后面的步骤就是在每个修改到进程用户页表的位置，都将相应的修改同步到进程内核页表中。一共要修改：fork()、exec()、growproc()、userinit()。

#### fork()
```c
// kernel/proc.c
int
fork(void)
{
  // ......

  // Copy user memory from parent to child. （调用 kvmcopymappings，将**新进程**用户页表映射拷贝一份到新进程内核页表中）
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0 ||
     kvmcopymappings(np->pagetable, np->kernelpgtbl, 0, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  // ......
}

```

#### exec()
```c
// kernel/exec.c
int
exec(char *path, char **argv)
{
  // ......

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));

  // 清除内核页表中对程序内存的旧映射，然后重新建立映射。
  uvmunmap(p->kernelpgtbl, 0, PGROUNDUP(oldsz)/PGSIZE, 0);
  kvmcopymappings(pagetable, p->kernelpgtbl, 0, sz);
  
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);
  // ......
}

```

#### growproc()
```c
// kernel/proc.c
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    uint64 newsz;
    if((newsz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
    // 内核页表中的映射同步扩大
    if(kvmcopymappings(p->pagetable, p->kernelpgtbl, sz, n) != 0) {
      uvmdealloc(p->pagetable, newsz, sz);
      return -1;
    }
    sz = newsz;
  } else if(n < 0){
    uvmdealloc(p->pagetable, sz, sz + n);
    // 内核页表中的映射同步缩小
    sz = kvmdealloc(p->kernelpgtbl, sz, sz + n);
  }
  p->sz = sz;
  return 0;
}
```

#### userinit()
对于 init 进程，由于不像其他进程，init 不是 fork 得来的，所以需要在 userinit 中也添加同步映射的代码。
```c
// kernel/proc.c
void
userinit(void)
{
  // ......

  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;
  kvmcopymappings(p->pagetable, p->kernelpgtbl, 0, p->sz); // 同步程序内存映射到进程内核页表中

  // ......
}
```

到这里，两个页表的同步操作就都完成了。

### 替换 copyin、copyinstr 实现

```c
// kernel/vm.c

// 声明新函数原型
int copyin_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len);
int copyinstr_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max);

// 将 copyin、copyinstr 改为转发到新函数
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}

int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

运行 grade：

```
pte printout: OK (4.8s) 
== Test answers-pgtbl.txt == answers-pgtbl.txt: OK 
== Test count copyin == 
$ make qemu-gdb
count copyin: OK (1.1s) 
== Test usertests == 
$ make qemu-gdb
(141.1s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
== Test time == 
time: OK 
Score: 66/66
```

# Optional challenges

* Use super-pages to reduce the number of PTEs in page tables.（跳过）
* Extend your solution to support user programs that are as large as possible; that is, eliminate the restriction that user programs be smaller than PLIC.（跳过）
* Unmap the first page of a user process so that dereferencing a null pointer will result in a fault. You will have to start the user text segment at, for example, 4096, instead of 0.（跳过）