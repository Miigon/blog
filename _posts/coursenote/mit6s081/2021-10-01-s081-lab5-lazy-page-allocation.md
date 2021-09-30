---
title: "[mit6.s081] 笔记 Lab5: Lazy Page Allocation | 内存页懒分配"
date: 2021-10-01 06:26:00 +0800
categories: [Course Notes, MIT6.S081]
tags: [operating system]
---
> 这是我自学 MIT6.S081 操作系统课程的 lab 代码笔记第五篇：Lazy page allocation。此 lab 大致耗时：5小时。  
> 
> 课程地址：[https://pdos.csail.mit.edu/6.S081/2020/schedule.html](https://pdos.csail.mit.edu/6.S081/2020/schedule.html)  
> Lab 地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/lazy.html](https://pdos.csail.mit.edu/6.S081/2020/labs/lazy.html)  
> 我的代码地址：[https://github.com/Miigon/my-xv6-labs-2020/tree/lazy](https://github.com/Miigon/my-xv6-labs-2020/tree/lazy)  
> Commits: [https://github.com/Miigon/my-xv6-labs-2020/commits/lazy](https://github.com/Miigon/my-xv6-labs-2020/commits/lazy)  
> 
> 本文中代码注释是编写博客的时候加入的，原仓库中的代码可能缺乏注释或代码不完全相同。  

# Lab 5: Lazy Page Allocation

> One of the many neat tricks an O/S can play with page table hardware is lazy allocation of user-space heap memory. Xv6 applications ask the kernel for heap memory using the sbrk() system call. In the kernel we've given you, sbrk() allocates physical memory and maps it into the process's virtual address space. It can take a long time for a kernel to allocate and map memory for a large request. Consider, for example, that a gigabyte consists of 262,144 4096-byte pages; that's a huge number of allocations even if each is individually cheap. In addition, some programs allocate more memory than they actually use (e.g., to implement sparse arrays), or allocate memory well in advance of use. To allow sbrk() to complete more quickly in these cases, sophisticated kernels allocate user memory lazily. That is, sbrk() doesn't allocate physical memory, but just remembers which user addresses are allocated and marks those addresses as invalid in the user page table. When the process first tries to use any given page of lazily-allocated memory, the CPU generates a page fault, which the kernel handles by allocating physical memory, zeroing it, and mapping it. You'll add this lazy allocation feature to xv6 in this lab.

实现一个内存页懒分配机制，在调用 sbrk() 的时候，不立即分配内存，而是只作记录。在访问到这一部分内存的时候才进行实际的物理内存分配。

本次 lab 分为三个部分，但其实都是属于同一个实验的不同步骤，所以本文将三点集合到一起：

Eliminate allocation from sbrk() (easy)
Lazy allocation (moderate)
Lazytests and Usertests (moderate)

## Lazy allocation & Tests

首先修改 sys_sbrk，使其不再调用 growproc()，而是只修改 p->sz 的值而不分配物理内存。

```c
// kernel/sysproc.c
uint64
sys_sbrk(void)
{
  int addr;
  int n;
  struct proc *p = myproc();
  if(argint(0, &n) < 0)
    return -1;
  addr = p->sz;
  if(n < 0) {
    uvmdealloc(p->pagetable, p->sz, p->sz+n); // 如果是缩小空间，则马上释放
  }
  p->sz += n; // 懒分配
  return addr;
}
```

修改 usertrap 用户态 trap 处理函数，为缺页异常添加检测，如果为缺页异常（`(r_scause() == 13 || r_scause() == 15)`），且发生异常的地址是由于懒分配而没有映射的话，就为其分配物理内存，并在页表建立映射：

```c
// kernel/trap.c

//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  // ......

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    uint64 va = r_stval();
    if((r_scause() == 13 || r_scause() == 15) && uvmshouldtouch(va)){ // 缺页异常，并且发生异常的地址进行过懒分配
      uvmlazytouch(va); // 分配物理内存，并在页表创建映射
    } else { // 如果不是缺页异常，或者是在非懒加载地址上发生缺页异常，则抛出错误并杀死进程
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
  }

  // ......

}
```

uvmlazytouch 函数负责分配实际的物理内存并建立映射。懒分配的内存页在被 touch 后就可以被使用了。  
uvmshouldtouch 用于检测一个虚拟地址是不是一个需要被 touch 的懒分配内存地址，具体检测的是：
1. 处于 `[0, p->sz)`地址范围之中（进程申请的内存范围）
2. 不是栈的 guard page（具体见 xv6 book，栈页的低一页故意留成不映射，作为哨兵用于捕捉 stack overflow 错误。懒分配不应该给这个地址分配物理页和建立映射，而应该直接抛出异常）  
    （解决 usertests 中的 stacktest 失败的问题）
3. 页表项不存在

```c
// kernel/vm.c

// touch a lazy-allocated page so it's mapped to an actual physical page.
void uvmlazytouch(uint64 va) {
  struct proc *p = myproc();
  char *mem = kalloc();
  if(mem == 0) {
    // failed to allocate physical memory
    printf("lazy alloc: out of memory\n");
    p->killed = 1;
  } else {
    memset(mem, 0, PGSIZE);
    if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
      printf("lazy alloc: failed to map page\n");
      kfree(mem);
      p->killed = 1;
    }
  }
  // printf("lazy alloc: %p, p->sz: %p\n", PGROUNDDOWN(va), p->sz);
}

// whether a page is previously lazy-allocated and needed to be touched before use.
int uvmshouldtouch(uint64 va) {
  pte_t *pte;
  struct proc *p = myproc();
  
  return va < p->sz // within size of memory for the process
    && PGROUNDDOWN(va) != r_sp() // not accessing stack guard page (it shouldn't be mapped)
    && (((pte = walk(p->pagetable, va, 0))==0) || ((*pte & PTE_V)==0)); // page table entry does not exist
}

```

由于懒分配的页，在刚分配的时候是没有对应的映射的，所以要把一些原本在遇到无映射地址时会 panic 的函数的行为改为直接忽略这样的地址。

uvmummap()：取消虚拟地址映射

```c
// kernel/vm.c
// 修改这个解决了 proc_freepagetable 时的 panic
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0) {
      continue; // 如果页表项不存在，跳过当前地址 （原本是直接panic）
    }
    if((*pte & PTE_V) == 0){
      continue; // 如果页表项不存在，跳过当前地址 （原本是直接panic）
    }
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}

```

uvmcopy()：将父进程的页表以及内存拷贝到子进程

```c
// kernel/vm.c
// 修改这个解决了 fork 时的 panic
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      continue; // 如果一个页不存在，则认为是懒加载的页，忽略即可
    if((*pte & PTE_V) == 0)
      continue; // 如果一个页不存在，则认为是懒加载的页，忽略即可
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

copyin() 和 copyout()：内核/用户态之间互相拷贝数据

由于这里可能会访问到懒分配但是还没实际分配的页，所以要加一个检测，确保 copy 之前，用户态地址对应的页都有被实际分配和映射。

```c
// kernel/vm.c
// 修改这个解决了 read/write 时的错误 (usertests 中的 sbrkarg 失败的问题)
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  if(uvmshouldtouch(dstva))
    uvmlazytouch(dstva);

  // ......

}

int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  if(uvmshouldtouch(srcva))
    uvmlazytouch(srcva);

  // ......

}
```

至此修改完成，在 xv6 中运行 lazytests 和 usertests 都应该能够成功了。  
如果在某一步出现了 remap 或者 leaf 之类的 panic，可能是由于页表项没有释放干净。可以从之前 pgtbl 实验中借用打印页表的函数 vmprint 的代码，并在可能有关的系统调用中打出，方便对页表进行调试。

> tip. 如果 usertests 某一步失败了，可以用 `usertests [测试名称]` 直接单独运行某个之前失败过的测试，例如 `usertests stacktest` 可以直接运行栈 guard page 的测试，而不用等待其他测试漫长的运行。
