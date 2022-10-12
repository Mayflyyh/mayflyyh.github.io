---
title: '6.S081 Lab: page tables'
date: 2022-09-23 00:49:32
categories:
- 6.S081
tags:
- OS
- lab
---

继续做实验..

<!-- more -->

In this lab you will explore page tables and modify them to simplify the functions that copy data from user space to kernel space.

Before you start coding, read Chapter 3 of the [xv6 book](https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf), and related files:

- `kern/memlayout.h`, which captures the layout of memory.
- `kern/vm.c`, which contains most virtual memory (VM) code.
- `kernel/kalloc.c`, which contains code for allocating and freeing physical memory.

## Print a page table

在 `vm.c` 中添加一个

```c
void _vmprint(pagetable_t pagetable,int deep){
    int i,j;
    for(i = 0; i < 512; i++){
        pte_t pte = pagetable[i];
        if(pte & PTE_V){
            for(j=0;j<deep;++j){
                printf("..");
                if(j!=deep-1)
                  printf(" ");
            }
            printf("%d: pte %p pa %p\n",i,pte,PTE2PA(pte));
            uint64 child = PTE2PA(pte);
            if(deep<3)
              _vmprint((pagetable_t)child,deep+1);
        }
    }
}

void vmprint(pagetable_t pagetable){
    printf("page table %p\n",pagetable);
    _vmprint(pagetable,1);
}
```

然后在 `exec.c` 中添加

```c
  if(p->pid == 1){
    vmprint(p->pagetable);
  }
```

![image-20220926101803456](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220926101803456.png)

课后问题

> Explain the output of `vmprint` in terms of Fig 3-4 from the text. What does page 0 contain? What is in page 2? When running in user mode, could the process read/write the memory mapped by page 1?



## A kernel page table per process

> xv6 有一个单独的 kernel page table，并且它是直接映射到物理地址中的。
>
> xv6 还为每个进程的 user address space 维护了单独的页表，只包含进程的 user memory 的映射。因为 kernel page table 不包含这些映射，user address 在 kernel 中是无效的。所以当 kernel 想要使用 system call 传递的 user pointer 时，kernel 首先要将这个指针转换成物理地址。
> 
> 本节和下节的目标是允许 kernel 直接使用 user pointer

本节任务是需要修改 kernel ，所以每个进程在处于 kernel 执行时 会使用这个进程自己的 kernel 。

1. 修改 `struct proc`，为每个进程维护一个 `kernel page`
2. 修改 `scheduler` 使得在切换进程时可以切换 kernel page
3. 每个进程的 kernel page 需要映射到全局的 kernel page table 上

在 `proc.c` 的 `struct proc` 中添加

```c
pagetable_t kpgtbl;      // Kernel page table
```

在 `vm.c` 中添加一个 `proc_kpgtbl()`，并将 `proc_kpgtbl()` 添加到 `allocproc()` 函数中，便于为每个进程都生成一个内核页表（通过修改 `kvminit` 产生）。

```c
pagetable_t proc_kpgtbl(){
    pagetable_t kpgtbl;
    if((kpgtbl=uvmcreate())==0){
        panic("proc_kpgtbl");
        return 0;
    }
  
        // uart registers
        if(mappages(kpgtbl, UART0, PGSIZE, UART0, PTE_R | PTE_W) != 0)
            panic("proc_kpgtbl");

        // virtio mmio disk interface
        if(mappages(kpgtbl, VIRTIO0, PGSIZE, VIRTIO0, PTE_R | PTE_W) != 0)
            panic("proc_kpgtbl");
        
        // CLINT
        if(mappages(kpgtbl, CLINT, 0x10000, CLINT, PTE_R | PTE_W) != 0)
            panic("proc_kpgtbl");

        // PLIC
        if(mappages(kpgtbl, PLIC, 0x400000, PLIC, PTE_R | PTE_W) != 0)
            panic("proc_kpgtbl");

        // map kernel text executable and read-only.
        if(mappages(kpgtbl, KERNBASE, (uint64)etext-KERNBASE, KERNBASE, PTE_R | PTE_X) != 0)
            panic("proc_kpgtbl");

        // map kernel data and the physical RAM we'll make use of.
        if(mappages(kpgtbl, (uint64)etext, PHYSTOP-(uint64)etext, (uint64)etext, PTE_R | PTE_W) != 0)
            panic("proc_kpgtbl");

        // map the trampoline for trap entry/exit to
        // the highest virtual address in the kernel.
        if(mappages(kpgtbl, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) != 0)
            panic("proc_kpgtbl");
    return kpgtbl;
}
```

在 `scheduler()` 中添加，使得 CPU 可以使用进程的内核页表

```c
w_satp(MAKE_SATP(p->kpgtbl));
sfence_vma();
```

同时要修改 `kvmpa()` ，这个函数翻译内核的虚拟地址到物理地址，之前是利用全局内核翻译，现在要修改成利用每个进程自己的内核页表翻译

```c
uint64
kvmpa(uint64 va)
{
  uint64 off = va % PGSIZE;
  pte_t *pte;
  uint64 pa;
  
//   pte = walk(kernel_pagetable, va, 0);
  pte = walk(myproc()->kpgtbl, va, 0);
  if(pte == 0)
    panic("kvmpa");
  if((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa+off;
}
```

释放进程时释放进程的内核页表 `proc_freekpgtbl()`，但不释放除了`kstack`以外的页物理内存页

```c
void proc_freekpgtbl(pagetable_t kpgtbl){
    // uvmunmap()
    // mappages(kpgtbl, UART0, PGSIZE, UART0, PTE_R | PTE_W) 

    uvmunmap(kpgtbl,UART0,1,0);
    // if(mappages(kpgtbl, VIRTIO0, PGSIZE, VIRTIO0, PTE_R | PTE_W) != 0)

    uvmunmap(kpgtbl,VIRTIO0,1,0);
    // if(mappages(kpgtbl, CLINT, 0x10000, CLINT, PTE_R | PTE_W) != 0)

    uvmunmap(kpgtbl,CLINT,0x10000/(PGSIZE),0);
    // if(mappages(kpgtbl, PLIC, 0x400000, PLIC, PTE_R | PTE_W) != 0)

    uvmunmap(kpgtbl,PLIC,0x400000/(PGSIZE),0);
    // if(mappages(kpgtbl, KERNBASE, (uint64)etext-KERNBASE, KERNBASE, PTE_R | PTE_X) != 0)

    uvmunmap(kpgtbl,KERNBASE,((uint64)etext-KERNBASE)/(PGSIZE),0);
    // if(mappages(kpgtbl, (uint64)etext, PHYSTOP-(uint64)etext, (uint64)etext, PTE_R | PTE_W) != 0)

    uvmunmap(kpgtbl,(uint64)etext,(PHYSTOP-(uint64)etext)/(PGSIZE),0);
    // if(mappages(kpgtbl, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) != 0)
    
    uvmunmap(kpgtbl,TRAMPOLINE,1,0);
    
    uvmunmap(kpgtbl,KSTACK(0),1,1);
    
    freewalk(kpgtbl);
}
```



## Simplify `copyin/copyinstr` 

> 之前因为只有全局内核，所以如果内核想要访问用户提供的指针指向的内存，就必须使用 process 的 page table 来翻译 process 提供的虚拟地址。
>
> 这一部分的工作是将 user mappings 添加到 process 的 kernel 中，以便于 `copyin/copyinstr` 可以直接解引用 user pointer

Replace the body of `copyin` in `kernel/vm.c` with a call to `copyin_new` (defined in `kernel/vmcopyin.c`); do the same for `copyinstr` and `copyinstr_new`. Add mappings for user addresses to each process's kernel page table so that `copyin_new` and `copyinstr_new` work. You pass this assignment if `usertests` runs correctly and all the `make grade` tests pass.

这样做是依赖于用户虚拟地址和kernel指令和数据的虚拟地址并不重合，xv6 的用户空间的虚拟地址从0开始，而 kernel 的内存从高地址开始。

但是这么做也会限制了用户空间的最大大小必须低于 `kernel` 虚拟地址的最低位。在 `kernel` 启动以后，xv6 的kernel 的最低的虚拟地址在 `0xC000000` ，是 `PLIC` 寄存器的地址。所以就要修改 xv6 避免用户进程的地址增长到 `PLIC` 寄存器的地址。

为什么说这里 kernel 的最低的虚拟地址在 `0xC000000` ？

因为 0x00001000 的 boot ROM 和 0x02000000 的 `CLINT` 只在内核启动时需要用到，之后不会再需要访问了。



`proc_tokpgtbl` 函数，将映射  `virtual address -> physical address  ` in user page table 在 kernel page table 也做一遍。

注意传入时要传入 `oldva` 与 `newva`，并一同向上取整(`PGROUNDUP`)，以避免内存不对齐产生的影响。

注意获取到 flag 后 要将 `PTE_U` 这个 flag 抹去，因为 kernel 不能直接访问带有 `PTE_U` 的 PTE。

```c
void proc_tokpgtbl(pagetable_t user, pagetable_t kpgtbl, uint64 oldva ,uint64 newva){
    oldva=PGROUNDUP(oldva);
    newva=PGROUNDUP(newva);
    if(newva>PLIC) 
      panic("proc_tokpgtbl");
    uint64 sz ;
    uint64 va;
    pte_t *pte;
    uint64 pa,der;
    uint flags;
    sz = newva-oldva;
    for(va = oldva, der = 0 ; der < sz; der += PGSIZE, va += PGSIZE){
        if((pte = walk(user, va, 0)) == 0)
            panic("uvmcopy: pte should exist");
        if((*pte & PTE_V) == 0)
            panic("uvmcopy: page not present");
        pa = PTE2PA(*pte);
        flags = PTE_FLAGS(*pte) & (~PTE_U);
        if(mappages(kpgtbl, va, PGSIZE, pa, flags) != 0){
            panic("usertokpgtbl");
        }
    }
}
```

解除映射，也要注意内存对齐的问题

```c
void proc_freetokpgtbl(pagetable_t kpgtbl, uint64 oldva, uint64 newva){
    oldva = PGROUNDUP(oldva);
    newva = PGROUNDUP(newva);
    uvmunmap(kpgtbl, newva, (oldva-newva)/(PGSIZE),0);
}
```

在 `fork()` 中加入

```c
  proc_tokpgtbl(np->pagetable,np->kpgtbl,np->sz);
```

因为 `fork()` 是对父进程的复制，可以知道父进程 `p->sz < PLIC`，所以有 `np->sz < PLIC `，此处不用再做判断。

在 `exec()` 中加入

```c
if(sz>(PLIC))
    goto bad;

proc_freetokpgtbl(p->kpgtbl,oldsz,0);
proc_tokpgtbl(p->pagetable,p->kpgtbl,0, p->sz);
```

删除 kernel page 对原本 pagetable 的映射，然后再做一次 kernel page 对当前 pagetable 的映射。

然后就是 `sbrk`，在 `growproc()` 中改写为

```c
 int
growproc(int n)
{
  uint sz,oldsz;
  struct proc *p = myproc();

  if((n+p->sz)>PLIC)
    return -1;

  oldsz = sz = p->sz;
  if(n > 0){
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
    proc_tokpgtbl(p->pagetable,p->kpgtbl,oldsz,sz);
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
    proc_freetokpgtbl(p->kpgtbl,oldsz,sz);
  }
  p->sz = sz;
  sz = oldsz;

  return 0;
}
```

注意这里一定要是增长或减少多少就做多少映射，不要浪费（之前做的是先把原本大小 sz 的映射全部撤销，然后重新对新的 sz 做一次映射），测试的强度还是很高的，如果有过多浪费就会 TLE。

当内核会用 `copyin ` 与 `copyinstr` 使用 `copyin_new` 和 `copyinstr_new` 代替原本的函数，使得翻译地址的工作在硬件级完成，加速虚拟地址翻译。


![image-20220925210905572](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220925210905572.png)

![image-20220925210933641](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220925210933641.png)




![image-20220926215430334](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220926215430334.png)
