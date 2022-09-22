---
title: xv6 source code analysis
date: 2022-09-20 20:33:51
tags:
- xv6

---

xv6手册笔记与源码分析

<!-- more -->

[TOC]

# Operating system interfaces

TO DO

# Operating system organizaiton

Thus an operating system must fulfill three requirements: multiplexing, isolation, and interaction.

Xv6 runs on a multi-core RISC-V microprocessor, and much of its low-level functionality (for example, its process implementation) is specific to RISC-V. RISC-V is a 64-bit CPU, and xv6 is written in “LP64” C, which means long (L) and pointers (P) in the C programming language are 64 bits, but int is 32-bit.

附 [中文RISC-V手册](http://riscvbook.com/chinese/RISC-V-Reader-Chinese-v2p1.pdf)

## Abstracting physical resources

为什么要有操作系统？操作系统管理下层硬件，并为上层提供服务。

为了实现强隔离，必须禁止应用程序直接访问硬件资源，而是利用操作系统提供的服务。比如UNIX应用程序只能用open，read，write和close与存储系统交互，而不能直接访问磁盘。操作系统提供了一个文件的抽象。

UNIX也通过这种方式提供CPU资源

##  User mode, supervisor mode, and system calls

RISC-V 有三种模式，machine mode, supervisor mode, and user mode

machine mode 有全部的权限，CPU启动时处于machine mode。machine mode主要用于配置计算机。xv6在machine mode下执行几行后就会切换到supervisor mode

In supervisor mode the CPU is allowed to execute privileged instructions:

- enabling and disabling interrupts
- reading and writing the register that holds the address of a page table
- ...

CPU不会执行处于user mode的应用程序的privileged instruction。

应用程序如果想要调用内核函数，必须切换到内核态。CPU提供一个特别的指令可以从user mode切换到supervisor mode (RISC-V provides the ecall instruction for this purpose.)

如果切换到了supervisor mode，kernel 会验证参数是否合法。

|File  |Description      |
| ---- | ---- |
|bio.c |Disk block cache for the file system.|
|console.c |Connect to the user keyboard and screen. | 
|entry.S | Very first boot instructions. |
|exec.c| exec() system call. |
|file.c | File descriptor support. |
|fs.c | File system. |
|kalloc.c | Physical page allocator. |
|kernelvec.S | Handle traps from kernel, and timer interrupts. |
|log.c | File system logging and crash recovery. |
|main.c | Control initialization of other modules during boot. |
|pipe.c | Pipes. |
|plic.c |RISC-V interrupt controller. |
|printf.c | Formatted output to the console. |
|proc.c| Processes and scheduling. |
|sleeplock.c| Locks that yield the CPU. |
|spinlock.c| Locks that don’t yield the CPU. |
|start.c| Early machine-mode boot code. |
|string.c| C string and byte-array library. |
|swtch.S| Thread switching. |
|syscall.c| Dispatch(调度) system calls to handling function. |
|sysfile.c| File-related system calls. |
|sysproc.c | Process-related system calls. |
|trampoline.S |Assembly code to switch between user and kernel. |
|trap.c | C code to handle and return from traps and interrupts. |
|uart.c | Serial-port console device driver. |
|virtio_disk.c |Disk device driver. |
|vm.c| Manage page tables and address spaces.|

From this book’s perspective, microkernel and monolithic operating systems share many key ideas. They implement system calls, they use page tables, they handle interrupts, they support processes, they use locks for concurrency control, they implement a file system, etc. This book focuses on these core ideas.
Xv6 is implemented as a monolithic kernel, like most Unix operating systems. Thus, the xv6 kernel interface corresponds to the operating system interface, and the kernel implements the complete operating system. Since xv6 doesn’t provide many services, its kernel is smaller than some microkernels, but conceptually xv6 is monolithic.

## Code: xv6 organization

kernel 的 inter-module interfaces 定义在 `/kernel/defs.h`

![image-20220921183738227](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220921183738227.png)

## Process overview

Xv6 uses page tables (which are implemented by hardware) to give each process its own address space. The RISC-V page table translates (or “maps”) a virtual address (the address that an RISC-V instruction manipulates) to a physical address (an address that the CPU chip sends to main memory).

xv6为每个进程分别维护页表，定义为user's address space.

user's address space 从虚拟地址的0开始，先是指令和全局变量，后有user stack和heap（用于malloc）。用户如果有需要，可以对heap进行拓展。

RISCV只是用低39位在页表中寻找虚拟地址。xv6只是用39位中的38位。所以最大地址就是0x3fffffffff ，也就是MAXVA (kernel/riscv.h:363)

At the top of the address space xv6 reserves a page for a trampoline and a page mapping the process’s trapframe.

xv6主要用这两个页面切入切出kernel

The trampoline page contains the code to transition in and out of the kernel and mapping the trapframe is necessary to save/restore the state of the user process.

kernel维护了每个进程的状态。在`struct proc`中。

一个进程最重要的kernel state是page table, kernel stack, run state.

kernel可以挂起当前运行的状态，并且恢复另一个线程的状态。

每个进程有user stack和kernel stack，当进程执行用户指令时使用user stack，而kernel stack为空。当进入kernel时（system call or interrupt），kernel在kernel stack上执行代码。显然kernel stack是独立的。

一个进程可以通过调用`ecall`执行 system call。`ecall`可以提升硬件 privilege level 并且将 pc 切换到 a kernel-defined entry point. The code at the entry point switches to a kernel stack and executes the kernel instructions that implement the system call. 当kernel执行结束后，会调用`sret` 切换到 user stack 并且回到 user space 。

进程的线程也会在kernel中阻塞以等待I/O

## Code: starting xv6, the first process and system call

When the RISC-V computer powers on, it initializes itself and runs a boot loader which is stored in read-only memory. The boot loader loads the xv6 kernel into memory. 

Then, in machine mode, the CPU executes xv6 starting at `_entry (kernel/entry.S:7).` The RISC-V starts with paging hardware disabled: virtual addresses map directly to physical addresses.

loader将xv6 kernel载入在物理地址0x80000000. (0x0:0x80000000包含了I/O设备)

```c
// kernel/entry.S

		# qemu -kernel loads the kernel at 0x80000000
        # and causes each hart (i.e. CPU) to jump there.
        # kernel.ld causes the following code to
        # be placed at 0x80000000.
.section .text
.global _entry
_entry:
        # set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + (hartid * 4096)
        la sp, stack0
        li a0, 1024*4
        csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
        # jump to start() in start.c
        call start
spin:
        j spin
```

sp = stack0 + (hartid * 4096)

![image-20220921192342270](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220921192342270.png)

![image-20220921192744071](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220921192744071.png)

The instructions at _entry set up a stack so that xv6 can run C code. Xv6 declares space for an initial stack, stack0, in the file start.c (kernel/start.c:11). The code at _entry loads the stack pointer register sp with the address stack0+4096, the top of the stack, because the stack on RISC-V grows down. Now that the kernel has a stack, _entry calls into C code at start (kernel/start.c:21).

```c
// kernel/start.c
void
start()
{
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;
  x |= MSTATUS_MPP_S;
  w_mstatus(x);

  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  w_mepc((uint64)main);

  // disable paging for now.
  w_satp(0);

  // delegate all interrupts and exceptions to supervisor mode.
  w_medeleg(0xffff);
  w_mideleg(0xffff);
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
  w_pmpcfg0(0xf);

  // ask for clock interrupts.
  timerinit();

  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
  w_tp(id);

  // switch to supervisor mode and jump to main().
  asm volatile("mret");
}
```

The function start performs some configuration that is only allowed in machine mode, and then switches to supervisor mode. To enter supervisor mode, RISC-V provides the instruction `mret`. This instruction is most often used to return from a previous call from supervisor mode to machine mode. start isn’t returning from such a call, and instead sets things up as if there had been one: it sets the previous privilege mode to supervisor in the register mstatus, it sets the return address to main by writing main’s address into the register `mepc`, disables virtual address translation in supervisor mode by writing 0 into the page-table register satp, and delegates all interrupts and exceptions to supervisor mode.

Before jumping into supervisor mode, start performs one more task: it programs the clock chip to generate timer interrupts. With this housekeeping out of the way, start “returns” to supervisor mode by calling mret. This causes the program counter to change to main (kernel/main.c:11).



After main (kernel/main.c:11) initializes several devices and subsystems, it creates the first process by calling userinit (kernel/proc.c:233). The first process executes a small program written in RISC-V assembly, which makes the first system call in xv6. initcode.S (user/initcode.S:3) loads the number for the exec system call, SYS_EXEC (kernel/syscall.h:8), into register a7, and then calls ecall to re-enter the kernel.

```c
// Set up first user process.
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
```

The kernel uses the number in register `a7` in `syscall` (kernel/syscall.c:132) to call the desired system call. The system call table (kernel/syscall.c:107) maps `SYS_EXEC` to `sys_exec`, which the kernel invokes. As we saw in Chapter 1, exec replaces the memory and registers of the current process with a new program (in this case, /init). 

Once the kernel has completed `exec`, it returns to user space in the `/init` process. `init (user/init.c:15) `creates a new console device file if needed and then opens it as file descriptors 0, 1, and 2. Then it starts a shell on the console. The system is up.



##  Security Model

TO DO

# Page Table

TO DO

## Kernel address space

xv6 为每个进程维护一个页表，描述每个用户的用户地址空间，另外还维护一个描述内核地址空间的页表。

The kernel configures the layout of its address space to give itself access to physical memory and various hardware resources at predictable virtual addresses.

The file (kernel/memlayout.h) declares the constants for xv6’s kernel memory layout.

![](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220922232341836.png)

QEMU 模拟了计算机，在 RAM 的 0x80000000 处启动，并且至少执行到 0x88000000。

QEMU 向软件公开设备接口（0x80000000 以下的 memory-mapped control registers）。

内核可以通过读写特定的物理地址与这些设备交互。

内核使用 `direct mapping` 的方式获取 RAM 和 memory-mapped device registers ，也就是说他们的虚拟地址和物理地址是相等的。`Direct mapping` 简化了读写物理内存的内核代码。比如说当 `fork` 为子进程分配用户内存时，allocator 会直接返回内存的物理地址。当fork 把父进程用户内存拷贝给儿子时，fork 会直接把这个地址当成虚拟地址。

有一些内核的虚拟地址不会采用直接映射的方式：

- The trampoline page.

  它被映射到虚拟地址的顶部。用户页表也会有这个映射。

  可以发现包含 trampoline code 的物理页被映射到内核的虚拟地址了两次，一次在虚拟地址的顶部，一次是直接映射。

- The kernel stack pages. 

  Each process has its own kernel stack, which is mapped high so that below it xv6 can leave an unmapped guard page. The guard page’s PTE is invalid (i.e., PTE_V is not set), so that if the kernel overflows a kernel stack, it will likely cause an exception and the kernel will panic. Without a guard page an overflowing stack would overwrite other kernel memory, resulting in incorrect operation. A panic crash is preferable.

The kernel maps the pages for the trampoline and the kernel text with the permissions PTE_R and PTE_X，others is PTE_R and PTE_X

## Code: creating an address space

xv6 中主要操作地址空间和页表的代码都在 `vm.c`

一个 pagetable_t 要么是内存页，或者是进程的页表。

主要的函数有两个：

- `walk` ， 为虚拟地址找到对应的 PTE
- `mappages`，为新的映射安装 PTE

以 `kvm` 开头的函数操作用户页表，以 `uvm` 开头的函数操作用户页表，其他函数两者都可以操作。

`copyout` 和 `copyin` 用于从系统调用提供的虚拟地址，将内核的数据拷贝入或者拷贝出。

这些代码之所以在 `vm.c` ，是因为他们需要显式地转换地址，以便于找到对应的物理地址（没太明白）

在 Boot 靠前的部分，main 会调用 `kvminit`，使用 `kvmmake` 创造一个内核页面。这个调用发生在RISC-V开启页面之前，所以地址直接对应物理内存。`kvmmake` 会先分配一页物理页来作为页表页的根。然后它会调用 `kvmmap` 以安装内核所需要的 translations，包含了内核的指令和数据，直到 PHYSTOP 的物理内存，和实际设备的内存。

`kvmmap` 会调用 `mappages`，以为一个范围的虚拟地址装载到对应范围的物理地址的映射。It does this separately for each virtual address in the range, at page intervals.

对于每个虚拟地址，`mappages ` 调用 `walk` 去寻找地址对应的 PTE 的地址。然后初始化 PTE 。

`walk` 模仿 RISC-V 的硬件去寻找虚拟地址的 PTE 

```c
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];//获取PTE
    if(*pte & PTE_V) { // 如果存在，就替换pagetable
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0) 
        return 0;// 否则分配新的pagetable，然后装载在pagetable[PX(level, va)]（原来的）里
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

`kvminithart` 装载内核页表。它将根页表页的物理地址写入寄存器 `satp` 。在此之后CPU会用内核页表页转换地址。Since the kernel uses an identity mapping, the now virtual address of the next instruction will map to the right physical memory address.（不明白）

`procinit` 为每个进程分配 kernel stack。它将每个栈映射到KSTACK生成的虚拟地址，这为无效的堆栈保护页留下了空间。

RISC-V 的 CPU 在 TLB 缓存页表项。当 xv6 更换页表时，必须告诉 CPU 使得对应的缓存 TLB 项失效。

RISC-V 可以用`sfence.vma`刷新当前 CPU 的 TLB。xv6 在重新装载 `stap` 寄存器以后，在 `kvminithart` 中执行 `sfence.vma`，and in the trampoline code that switches to a user page table before returning to user space.

##  Physical memory allocation

内核需要在运行时为 page table, user memory, kernel stack, pipe buffers 分配或者释放物理内存。

xv6 使用 kernel 末端和 PHYSTOP 之间的物理内存作为运行时分配的内存。它一次会释放或者分配大小为 4096 bytes 的一页。xv6 使用一个链表维护空闲的页

## Code: Physical memory allocator

`kinit ` 会初始化 allocator，在这里 xv6 假设机器有 128M 的RAM。

`kfree` 会将要 free 的空间用 1 填充

## Process address space

用户进程从虚拟地址 0 开始，最多可达 $2^{38}$，也就是 256G。

进程地址空间包括：

- pages that contain the text of program
  - PTE_R, PTE_X, and PTE_U
- pages that contain pre-initialized data of program
  - PTE_R, PTE_W, and PTE_U
- a page for the stack
  - PTE_R, PTE_W, and PTE_U
  - the initial contents as created by exec.
  - Strings containing the command-line arguments, as well as an array of pointers to them, are at the very top of the stack
- pages for the heap
  - PTE_R, PTE_W, and PTE_U

xv6 通过在 stack 正下方放置了一个不能访问的守护页（清除了 `PTE_U` )，而现实的操作系统会自动申请更多的内存当用户栈溢出时。

当 xv6 申请更多的用户内存时，xv6 会增长它的用户栈。xv6 先使用 `kalloc` 分配物理内存，然后在进程的页表页添加指向新的物理页的 PTE 

Third, the kernel maps a page with trampoline code at the top of the user address space (without PTE_U), thus a single page of physical memory shows up in all address spaces, but can be used only by the kernel.

## Code: sbrk

TO DO

## Code: exec

TO DO



















































---



# xv6 启动过程

启动过程

##  main.c

```c
// kernel/main.c
#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "defs.h"

volatile static int started = 0;

// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode cache
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}  
```

### kinit()

```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```

`freerange()`

```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}
```

其中

```c
#define PGROUNDUP(sz)  (((sz)+PGSIZE-1) & ~(PGSIZE-1))
#define PGROUNDDOWN(a) (((a)) & ~(PGSIZE-1))
```

`PGROUNDUP(sz)` 表示大于等于sz，且是PGSIZE倍数的地址

`PGROUNDDOWN(sz)` 表示小于等于sz，且是PGSIZE倍数的地址

`kfree()` 的作用是释放地址`p`对应的page，然后将其载入空闲列表中

在`kinit()`中调用`freerange()`时的参数是 `end`(the end of kernel)  和 `PHYSTOP`(物理地址末端）

对应手册，

> 3.4 Physical memory allocation 
>
>The kernel must allocate and free physical memory at run-time for c
>
>Xv6 uses the physical memory between the end of the kernel and PHYSTOP for run-time allocation. It allocates and frees whole 4096-byte pages at a time. It keeps track of which pages are free by threading a linked list through the pages themselves. Allocation consists of removing a page from the linked list; freeing consists of adding the freed page to the list. 

可知此处分配的物理内存是用于 page tables, user memory, kernel stacks, and pipe buffers

### kvminit()

create the kernel's page table

```c
// kernel/vm.c

/*
 * create a direct-map page table for the kernel.
 */
void
kvminit()
{
  kernel_pagetable = (pagetable_t) kalloc();
  memset(kernel_pagetable, 0, PGSIZE);

  // uart registers
  kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}
```

注意到手册中说这里的地址是直接映射到物理内存的

>  This call occurs before xv6 has enabled paging on the RISC-V, so addresses refer directly to physical memory. 

`kernel_pagetable` 

- the root page-table page

`kalloc()` 就是从`kalloc()`初始化的页中获取一页

> Then it calls kvmmap to install the translations that the kernel needs. The translations include the kernel’s instructions and data, physical memory up to PHYSTOP, and memory ranges which are actually devices. 

这里手册并没有展开讲，应该是在后面会讲到。

虽然不清楚具体kvmmap的是什么，但还是可以分析一下kvmmap函数

#### kvmmap()

```c
// add a mapping to the kernel page table.
// only used when booting.
// does not flush TLB or enable paging.
void
kvmmap(uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(kernel_pagetable, va, sz, pa, perm) != 0)
    panic("kvmmap");
}
```

#### mappages()

```c
// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa. va and size might not
// be page-aligned. Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

14行先用`walk()`在页表中找到va对应的L0级的pte

18行为这个L0的PTE填写具体对应的物理地址

```c
#define PGSIZE 4096 // bytes per page
#define PGSHIFT 12  // bits of offset within a page

// extract the three 9-bit page table indices from a virtual address.
#define PXMASK          0x1FF // 9 bits
#define PXSHIFT(level)  (PGSHIFT+(9*(level)))
#define PX(level, va) ((((uint64) (va)) >> PXSHIFT(level)) & PXMASK)

#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10)

#define PTE2PA(pte) (((pte) >> 10) << 12)

#define PTE_FLAGS(pte) ((pte) & 0x3FF)
```

```c
// kernel/vm.c
// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

walk()返回L0级页表va所对应的项

结合手册中的图理解，先提取va中的L2，然后在pagetable中找到L2对应的PTE地址

将PTE中的数据右移10位再左移12位作为下一级pagetable，（PTE不存在就创建）

注意pagetable一直是物理地址

![pic1](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/20220920220342.png)

### kvminithart()

It writes the physical address of the root page-table page into the register satp

After this the CPU will translate addresses using the kernel page table. Since the kernel uses an identity mapping, the now virtual address of the next instruction will map to the right physical memory address.

```c
// Switch h/w page table register to the kernel's page table,
// and enable paging.
void
kvminithart()
{
  w_satp(MAKE_SATP(kernel_pagetable));
  sfence_vma();
}
```

将kernel_tabel写入satp寄存器

#### procinit()



