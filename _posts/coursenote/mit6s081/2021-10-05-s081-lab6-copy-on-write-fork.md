---
title: "[mit6.s081] 笔记 Lab6: Copy-on-write fork | fork 懒拷贝"
date: 2021-10-05 15:21:00 +0800
categories: [Course Notes, MIT6.S081]
tags: [operating system]
---
> 这是我自学 MIT6.S081 操作系统课程的 lab 代码笔记第六篇：Copy-on-write fork。此 lab 大致耗时：4小时。  
> 
> 课程地址：[https://pdos.csail.mit.edu/6.S081/2020/schedule.html](https://pdos.csail.mit.edu/6.S081/2020/schedule.html)  
> Lab 地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/cow.html](https://pdos.csail.mit.edu/6.S081/2020/labs/cow.html)  
> 我的代码地址：[https://github.com/Miigon/my-xv6-labs-2020/tree/cow](https://github.com/Miigon/my-xv6-labs-2020/tree/cow)  
> Commits: [https://github.com/Miigon/my-xv6-labs-2020/commits/cow](https://github.com/Miigon/my-xv6-labs-2020/commits/cow)  
> 
> 本文中代码注释是编写博客的时候加入的，原仓库中的代码可能缺乏注释或代码不完全相同。  

# Lab 6: Copy-on-write fork

COW fork() creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages. COW fork() marks all the user PTEs in both parent and child as not writable. When either process tries to write one of these COW pages, the CPU will force a page fault. The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable. When the page fault handler returns, the user process will be able to write its copy of the page.

COW fork() makes freeing of the physical pages that implement user memory a little trickier. A given physical page may be referred to by multiple processes' page tables, and should be freed only when the last reference disappears.

实现 fork 懒复制机制，在进程 fork 后，不立刻复制内存页，而是将虚拟地址指向与父进程相同的物理地址。在父子任意一方尝试对内存页进行修改时，才对内存页进行复制。
物理内存页必须保证在所有引用都消失后才能被释放，这里需要有引用计数机制。

## Implement copy-on write (hard)

> 为了便于区分，本文将只创建引用而不进行实际内存分配的页复制过程称为「懒复制」，将分配新的内存空间并将数据复制到其中的过程称为「实复制」

### fork 时不立刻复制内存

首先修改 uvmcopy()，在复制父进程的内存到子进程的时候，不立刻复制数据，而是建立指向原物理页的映射，并将父子两端的页表项都设置为不可写。

```c
// kernel/vm.c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    if(*pte & PTE_W) {
      // 清除父进程的 PTE_W 标志位，设置 PTE_COW 标志位表示是一个懒复制页（多个进程引用同个物理页）
      *pte = (*pte & ~PTE_W) | PTE_COW;
    }
    flags = PTE_FLAGS(*pte);
    // 将父进程的物理页直接 map 到子进程 （懒复制）
    // 权限设置和父进程一致
    // （不可写+PTE_COW，或者如果父进程页本身单纯只读非 COW，则子进程页同样只读且无 COW 标识）
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }
    // 将物理页的引用次数增加 1
    krefpage((void*)pa);
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}

```

> UPDATE 2023-02-20: 上述代码一开始的版本没有考虑只读页的拷贝，会导致单纯的非 COW 只读页被错误标记为 COW 页从而变成可写。  
> 这里给出的代码[已经修复该问题](https://github.com/Miigon/my-xv6-labs-2020/commit/c119f033881ffe19f4000d0149043e717304f659)，感谢 [@zztaki](https://github.com/zztaki) 指出该问题。  
> 修复后，只读页会直接共享物理页，并参与引用计数，但是不会被打上 COW 标记。

上面用到了 PTE_COW 标志位，用于标示一个映射对应的物理页是否是懒复制页。这里 PTE_COW 需要在 riscv.h 中定义：

```c
// kernel/riscv.h
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access
#define PTE_COW (1L << 8) // 是否为懒复制页，使用页表项 flags 中保留的第 8 位表示
// （页表项 flags 中，第 8、9、10 位均为保留给操作系统使用的位，可以用作任意自定义用途）
```

这样，fork 时就不会立刻复制内存，只会创建一个映射了。这时候如果尝试修改懒复制的页，会出现 page fault 被 usertrap() 捕获。接下来需要在 usertrap() 中捕捉这个 page fault，并在尝试修改页的时候，执行实复制操作。

### 捕获写操作并执行复制

与 lazy allocation lab 类似，在 usertrap() 中添加对 page fault 的检测，并在当前访问的地址符合懒复制页条件时，对懒复制页进行实复制操作：

```c
// kernel/trap.c
void
usertrap(void)
{

  // ......

  } else if((which_dev = devintr()) != 0){
    // ok
  } else if((r_scause() == 13 || r_scause() == 15) && uvmcheckcowpage(r_stval())) { // copy-on-write
    if(uvmcowcopy(r_stval()) == -1){ // 如果内存不足，则杀死进程
      p->killed = 1;
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  // ......

}
```

同时 copyout() 由于是软件访问页表，不会触发缺页异常，所以需要手动添加同样的监测代码（同 lab5），检测接收的页是否是一个懒复制页，若是，执行实复制操作：
```c
// kernel/vm.c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    if(uvmcheckcowpage(dstva)) // 检查每一个被写的页是否是 COW 页
      uvmcowcopy(dstva);
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    
    // .......memmove from src to pa0

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }

  // ......
}

```

> UPDATE 2023-02-20: 上述代码原始版本只检查了第一个目标页的 COW 状态，对于跨越多页的 copyout，如果目标页中有多个 COW 页，只有刚好在地址范围开头的第一个页会被检查，导致共享页被误写。
> 感谢 [@zztaki](https://github.com/zztaki) 指出该问题，这里给出的代码[已经修复该问题](https://github.com/Miigon/my-xv6-labs-2020/commit/c119f033881ffe19f4000d0149043e717304f659)。
> 该版本对每一个目标页，在写入之前都对 COW 标志位进行检查。

实现懒复制页的检测（`uvmcheckcowpage()`）与实复制（`uvmcowcopy()`）操作：

```c
// kernel/vm.c
// 检查一个地址指向的页是否是懒复制页
int uvmcheckcowpage(uint64 va) {
  pte_t *pte;
  struct proc *p = myproc();
  
  return va < p->sz // 在进程内存范围内
    && ((pte = walk(p->pagetable, va, 0))!=0)
    && (*pte & PTE_V) // 页表项存在
    && (*pte & PTE_COW); // 页是一个懒复制页
}

// 实复制一个懒复制页，并重新映射为可写
int uvmcowcopy(uint64 va) {
  pte_t *pte;
  struct proc *p = myproc();

  if((pte = walk(p->pagetable, va, 0)) == 0)
    panic("uvmcowcopy: walk");
  
  // 调用 kalloc.c 中的 kcopy_n_deref 方法，复制页
  // (如果懒复制页的引用已经为 1，则不需要重新分配和复制内存页，只需清除 PTE_COW 标记并标记 PTE_W 即可)
  uint64 pa = PTE2PA(*pte);
  uint64 new = (uint64)kcopy_n_deref((void*)pa); // 将一个懒复制的页引用变为一个实复制的页
  if(new == 0)
    return -1;
  
  // 重新映射为可写，并清除 PTE_COW 标记
  uint64 flags = (PTE_FLAGS(*pte) | PTE_W) & ~PTE_COW;
  uvmunmap(p->pagetable, PGROUNDDOWN(va), 1, 0);
  if(mappages(p->pagetable, va, 1, new, flags) == -1) {
    panic("uvmcowcopy: mappages");
  }
  return 0;
}
```

到这里，就已经确定了大体的逻辑了：在 fork 的时候不复制数据只建立映射+标记，在进程尝试写入的时候进行实复制并重新映射为可写。

接下来，还需要做页的生命周期管理，确保在所有进程都不使用一个页时才将其释放

### 物理页生命周期以及引用计数

在 kalloc.c 中，我们需要定义一系列的新函数，用于完成在支持懒复制的条件下的物理页生命周期管理。

在原本的 xv6 实现中，一个物理页的生命周期内，可以支持以下操作：
* kalloc(): 分配物理页
* kfree(): 释放回收物理页

而在支持了懒分配后，由于一个物理页可能被多个进程（多个虚拟地址）引用，并且必须在最后一个引用消失后才可以释放回收该物理页，所以一个物理页的生命周期内，现在需要支持以下操作：
* kalloc(): 分配物理页，将其引用计数置为 1
* krefpage(): 创建物理页的一个新引用，引用计数加 1
* kcopy_n_deref(): 将物理页的一个引用实复制到一个新物理页上（引用计数为 1），返回得到的副本页；并将本物理页的引用计数减 1
* kfree(): 释放物理页的一个引用，引用计数减 1；如果计数变为 0，则释放回收物理页

一个物理页 p 首先会被父进程使用 kalloc() 创建，fork 的时候，新创建的子进程会使用 krefpage() 声明自己对父进程物理页的引用。当尝试修改父进程或子进程中的页时，kcopy_n_deref() 负责将想要修改的页实复制到独立的副本，并记录解除旧的物理页的引用（引用计数减 1）。最后 kfree() 保证只有在所有的引用者都释放该物理页的引用时，才释放回收该物理页。

这里首先定义一个数组 pageref[] 以及对应的宏，用于记录与获取某个物理页的引用计数：

```c
// kernel/kalloc.c

// 用于访问物理页引用计数数组
#define PA2PGREF_ID(p) (((p)-KERNBASE)/PGSIZE)
#define PGREF_MAX_ENTRIES PA2PGREF_ID(PHYSTOP)

struct spinlock pgreflock; // 用于 pageref 数组的锁，防止竞态条件引起内存泄漏
int pageref[PGREF_MAX_ENTRIES]; // 从 KERNBASE 开始到 PHYSTOP 之间的每个物理页的引用计数
// note:  reference counts are incremented on fork, not on mapping. this means that
//        multiple mappings of the same physical page within a single process are only
//        counted as one reference.
//        this shouldn't be a problem, though. as there's no way for a user program to map
//        a physical page twice within it's address space in xv6.

// 通过物理地址获得引用计数
#define PA2PGREF(p) pageref[PA2PGREF_ID((uint64)(p))]

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&pgreflock, "pgref"); // 初始化锁
  freerange(end, (void*)PHYSTOP);
}

void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&pgreflock);
  if(--PA2PGREF(pa) <= 0) {
    // 当页面的引用计数小于等于 0 的时候，释放页面

    // Fill with junk to catch dangling refs.
    // pa will be memset multiple times if race-condition occurred.
    memset(pa, 1, PGSIZE);

    r = (struct run*)pa;

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  }
  release(&pgreflock);
}

void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r){
    memset((char*)r, 5, PGSIZE); // fill with junk
    // 新分配的物理页的引用计数为 1
    // (这里无需加锁)
    PA2PGREF(r) = 1;
  }
  
  return (void*)r;
}

// Decrease reference to the page by one if it's more than one, then
// allocate a new physical page and copy the page into it.
// (Effectively turing one reference into one copy.)
// 
// Do nothing and simply return pa when reference count is already
// less than or equal to 1.
// 
// 当引用已经小于等于 1 时，不创建和复制到新的物理页，而是直接返回该页本身
void *kcopy_n_deref(void *pa) {
  acquire(&pgreflock);

  if(PA2PGREF(pa) <= 1) { // 只有 1 个引用，无需复制
    release(&pgreflock);
    return pa;
  }

  // 分配新的内存页，并复制旧页中的数据到新页
  uint64 newpa = (uint64)kalloc();
  if(newpa == 0) {
    release(&pgreflock);
    return 0; // out of memory
  }
  memmove((void*)newpa, (void*)pa, PGSIZE);

  // 旧页的引用减 1
  PA2PGREF(pa)--;

  release(&pgreflock);
  return (void*)newpa;
}

// 为 pa 的引用计数增加 1
void krefpage(void *pa) {
  acquire(&pgreflock);
  PA2PGREF(pa)++;
  release(&pgreflock);
}
```

这里可以看到，为 pageref[] 数组定义了自旋锁 pgreflock，并且在除了 kalloc 的其他操作中，都使用了 `acquire(&pgreflock);` 和 `release(&pgreflock);` 获取和释放锁来保护操作的代码。这里的锁的作用是防止竞态条件（race-condition）下导致的内存泄漏。

举一个很常见的 fork() 后 exec() 的例子：
```
父进程: 分配物理页 p（p 引用计数 = 1）
父进程: fork()（p 引用计数 = 2）
父进程: 尝试修改 p，触发页异常
父进程: 由于 p 引用计数大于 1，开始实复制 p（p 引用计数 = 2）
--- 调度器切换到子进程
子进程: exec() 替换进程影像，释放所有旧的页
子进程: 尝试释放 p（引用计数减 1），子进程丢弃对 p 的引用（p 引用计数 = 1）
--- 调度器切换到父进程
父进程: （继续实复制p）创建新页 q，将 p 复制到 q，将 q 标记为可写并建立映射，在这过程中父进程丢弃对旧 p 的引用
```

在这一个执行流过后，最终结果是物理页 p 并没有被释放回收，然而父进程和子进程都已经丢弃了对 p 的引用（页表中均没有指向 p 的页表项），这样一来 p 占用的内存就属于泄漏内存了，永远无法被回收。

加了锁之后，保证了这种情况不会出现。

注意 kalloc() 可以不用加锁，因为 kmem 的锁已经保证了同一个物理页不会同时被两个进程分配，并且在 kalloc() 返回前，其他操作 pageref() 的函数也不会被调用，因为没有任何其他进程能够在 kalloc() 返回前得到这个新页的地址。

### 执行测试

```
$ make grade
......
== Test running cowtest == 
$ make qemu-gdb
(10.4s) 
== Test   simple == 
  simple: OK 
== Test   three == 
  three: OK 
== Test   file == 
  file: OK 
== Test usertests == 
$ make qemu-gdb
(99.8s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
== Test time == 
time: OK 
Score: 110/110
```

如果测试失败，可在 xv6 中手动执行 cowtest 以及 usertests 单独测试，并观察输出。
usertests 可能会输出以下错误：
```
FAILED -- lost some free pages 32442 (out of 32448)
```
该错误是 usertests 检测到运行前后的空闲页数量减少，也就是检测到发生了内存泄漏。检查上面的 kalloc.c 中的操作有没有正确加锁，或者一些页的分配/释放是否正确。