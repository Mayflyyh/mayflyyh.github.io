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

##  Paging hardware

RISCV kernel 和 user 指令都在虚拟地址操作，RISCV 通过页表硬件以将虚拟地址映射到物理地址。

xv6 runs on Sv39 RISC-V, which means that only the bottom 39 bits of a 64-bit virtual address are used; the top 25 bits are not used.

在 Sv39 的配置中，一个 pagetabe 包含了 $2^{27}=134217728$ 个 PTES。每个 PTE 包含了一个 44bit 的 physical page number (PPN) 和一些 flags。页表硬件通过 39 位的高 27 位找寻 PTE。 并且通过 PTE 中的 PPN (44bit) 加上 虚拟地址的低 12 位作为一个 56 位的物理地址。

实际上采用了 3 级树形 page table ，以不需要存储没有实际用到的 PTE

`PTE_V` 表示该 PTE 是否存在

`PTE_R` 表示是否允许指令读这个 page

`PTE_W` 表示是否允许指令写这个 page

`PTE_X` 表示是否允许 CPU 解释这个 page 的内容并且执行他们。

`PTE_U` 表示是否允许 `supervisor mode` 以外的模式读取这个 page（权限控制）

为了让硬件可以使用页表，kernel 必须将 root page-table page 的物理地址写入 `satp` 寄存器。每个 CPU 拥有自己的 `satp` 寄存器，CPU 将通过 `stap` 指向的 page 把接下来指令生成的地址进行翻译。每个 CPU 拥有自己的 `satp` 寄存器，所以每个 CPU 可以执行不同进程。

指令只会使用虚拟地址，每个虚拟地址由 page hardware 翻译成物理地址然后发送到 DRAM 硬件以读写。

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

`sbrk` 是 process 用来缩小或增长 memory 的系统调用。由 `growproc` 实现，`growproc` 调用 `uvmalloc` 和 `uvmdealloc`。

`uvmalloc` 通过 `kalloc` 分配物理内存，使用 `mappages` 向 user page table 中添加 PTES。

`vmdealloc` 调用 `uvmunmap`，以使用 `walk`  找寻 PTEs 并且释放与其有关的物理内存。

xv6 进程的 page table 不止告诉硬件如何映射虚拟地址，也作为分配给进程的物理内存页的唯一记录。这也是 user memory 需要遍历用户 page table的原因。

## Code: exec

`exec` 从文件系统中的文件初始化 the user part of an address space 。 `exec` 使用 `namei` 打开指定的二进制路径 `path`。然后他会读 `ELF header`。xv6 应用程序使用 ELF-format 描述，定义在 `kernel/elf.h`。

ELF 二进制文件由 `ELF header`，`struct elfhdr` 和 `struct proghdr` 构成。

每一个 `proghdr` 描述了应用程序必须载入内存的一段。xv6 程序只有一个 program section header ，但是其他系统可能有分别的指令与数据。

第一步，快速检查文件是否可能包含ELF二进制文件。一个 ELF 二进制文件起始于 四个字节的 `magic number`，0x7F , E , L , F , 或者是 `0x464C457FU` ("\x7FELF" in little endian) 。如果 ELF 头有正确`magic number`，`exec` 会认为它是正确的 ELF 文件



















































---



# Traps and system calls

## Calling Convention

long double 128-bit

C types `char` and `unsigned char` are zero-extended when stored in a RISC-V integer register.

All `unsigned` are zero-extended, otherwise are sign-extended.

RV64 的C编译器和兼容软件都保持对数据类型的内存对齐。

`a0-a7` integer registers，`fa0-fa7` floating-point registers

When primitive arguments twice the size of a pointer-word are passed on the stack, they are naturally aligned. When they are passed in the integer registers, they reside in an aligned even-odd register pair, with the even register holding the least-significant bits.

当两倍于指针字大小的参数传入寄存器时，他们会利用一对偶奇寄存器，偶寄存器存储 least-significant bits.

`void foo(int, long long)` is passed its first argument in a0 and its second in a2 and a3. Nothing is passed in a1.

## Start

有三种使得 CPU 跳出普通指令的执行，强制跳转到一段特别的代码去执行事项。

第一种情况是 `system call` ，用户程序调用 `ecall` 使得内核为它做一些事。

第二种情况是 `exception`，当指令做了一些不合法的事情时。

第三种情况是 `device interrupt`，当一个设备触发了提示它需要被注意。

本书用 `trap` 代指以上情况。`trap` 对指令应该是透明的。值得注意的是中断代码不期望被中断。

`trap` 强制将控制权转入内核的过程通常是这样的：

1. kernel 保存需要恢复的寄存器和执行的其他状态
2. kernel 执行正确的 `handler code`
3. kernel 恢复保存的寄存器和状态，从 `trap` 恢复。

xv6 trap handling 分为四个阶段：

1. hardware actions taken by the RISC-V CPU
2. an assembly “vector” that prepares the way for kernel C code
3. a C trap handler that decides what to do with the trap
4. the system call or device-driver service routine. 

尽管可以用一个代码路径处理三个不同类型的 trap，但是事实证明对于：

1. traps from user space
2. traps from kernel space
3. time interrupts

这三种情况分别准备 assembly vectors and C trap handlers 更好。

## RISC-V trap machinery

每一个 RISC-V CPU 都有一组控制寄存器，kernel 通过写这些寄存器告诉 CPU 怎么处理这些 trap，kernel 也可以读寄存器来发现已经发生的 trap。可以在 `riscv.h` 中看见更多寄存器。

An outline of the most important registers:

- `stvec`：内核将 trap handler 的地址写在这里。RISC-V 跳转到这里处理 trap

- `sepc`：当 trap 发生时，RISC-V 将 PC 保存在这里。（因为 `pc` 会被 `stvec` 重写）。`sret` (return from trap) 会将 `sepc` 考入到 `pc` 。The kernel can write to `sepc` to control where `sret` goes.

- `scause`：RISC-V 将 trap 的原因写入在此。

- `sscratch`：The kernel places a value here that comes in handy at the very start of a trap handler.

- `sstatus`：

  - The `SIE` bit in `sstatus` controls whether device interrupts are enabled. If the kernel clears `SIE`, the RISC-V will defer device interrupts until the kernel sets `SIE`. 
  - The `SPP` bit indicates whether a trap came from user mode or supervisor mode, and controls to what mode `sret` returns.

  The above registers relate to traps handled in supervisor mode, and they cannot be read or written in user mode.

  在 machine mode 下有一组等效的寄存器组，xv6 只使用它们用于处理时钟中断。

  当需要执行 trap 时，RISC-V 会对所有类型执行以下步骤（除了时钟中断）：

  1. 如果 trap 是 device interrupt 并且 `sstatus` 的 `SIE` 位被清除了，不执行以下过程。
  2. 清除 `SIE` 以关闭中断
  3. 将 `pc` 拷贝到 `sepc`
  4. 设置 `sstatus` 中的 `SPP` 位以保存当前的状态 (user or supervisor)
  5. 设置 `scause` 以反映 trap 的原因
  6. 将状态设置为 `supervisor`
  7. 将 `stvec` 拷贝到 `pc`
  8. 开始从新 `pc` 执行

注意 CPU 不会切换到内核页表，不会切换到内核栈，除了 `pc` 以外不会保存任何寄存器。Kernel 必须完成这些任务。

（CPU只会做最小的工作以提供灵活性和性能）

## Traps from user space

来自用户空间的高级别的 trap 路线是通过 `userrvec`，`usertrap`，返回时是 `usertrapret` 和 `uesrret`。

从用户代码 trap 比从内核更复杂，因为指向用户页表的`satp`并不映射内核，且指针可能包含不合法甚至于有害的值。

因为 RISC-V 硬件不在 trap 时切换页表，所以用户页表必须包含对 `userrvec` 的映射和 `stvec` 所指向的 trap vector  instruction. `userrvec` 必须切换 `stap` 以指向内核页表。为了在切换后继续执行指令，`userrvec` 必须在内核页表和用户页表的相同地址映射。

xv6 通过在 `trampoline` 包含 `userrvec` 以满足这些限制。xv6 映射 `trampoline` 在用户页表和内核页表的相同虚拟地址（这个虚拟地址是`TRAMPOLINE`）。`trampoline`的内容被设置在`trampoline.S`。在执行用户代码时，`stvec` 被设置为 `userrvec`。

`userrvec` 开始后，32个寄存器会被中断代码占用。但是 `userrvec` 需要修改一些寄存器，以设置 `satp` 并且生成保存寄存器的地址。`csrrw` 指令 在 `userrvec` 开始时交换`a0`和 `sscratch` 寄存器的内容。现在 `userrvec` 可以使用 `a0` 寄存器。

 `userrvec` 下一步需要保存用户寄存器。在进入用户空间前，内核设置`sscratch` 指向每个进程的`trapframe`（保存所有用户寄存器的空间）。因为 `satp` 仍引用用户页表，`userrvec` 需要 `trapframe` 被映射在用户空间。当创建新进程时，xv6 会为进程的 `trapframe` 分配一页，并将它映射到虚拟地址的 `TRAPFRAME` 处，且总是在 `TRAMPOLINE` 下面。进程的 `p->trapframe` 总是指向 trapframe，因为是它的物理地址，所以内核可以通过内核页表访问它。

在交换了 `a0` 和 `sscratch` 之后，`a0` 持有一个指向当前进程 trapframe 的指针。`userrvec` 现在保存所有寄存器在此处，包括从 `sscratch `读取的用户的 `a0`。

`trapframe` 保存指向当前进程的内核栈的指针，当前CPU的hartid，和 `usertrap` 的地址和内核页表的地址。`userrvec` 检索这些值，将 `satp` 切换到内核页表，并调用 `usertrap`.

`usertrap` 的工作是确定 trap 的原因，处理它然后返回 (在 `trap.c`) 。它先改变 `stvec`，使得内核中的 trap 可以被 `kernelvec` 处理。它需要保存 `sepc` ，因为 `usertrap` 中可能出现进程切换导致 `spec` 被覆盖。

如果 trap 是一个系统调用，那么 `syscall` 会处理它。

如果 trap 是一个设备终端，那么 `devintr` 会处理它。

否则 trap 是一个异常，内核会杀死出错的进程。 

系统调用会为保存的用户 `pc` 上加4，以使得 pc 指针指向 `ecall` 指令。在退出时，`usertrap` 会检查进程是否已被杀死，或者在 `trap` 是时钟中断时释放 CPU。

返回用户空间的第一步是调用 `usertrapret` 。这个函数设置 RISCV 控制寄存器以为来自用户空间的未来的 trap 做准备。这包括更改 `stvec` 以引用 `uservec`，准备 `uservec` 所依赖的 trapframe fields，将 `sepc` 设置为之前保存的 `pc`。结束时，`usertrapret` 调用 映射到用户和内核空间的 trampoline page 上的 `userret` ，`userret` 中的汇编代码将切换页表。

`usertrapret` 对 `userret` 的调用传递了一个指向 `a0` 中的用户页表和 `a1` 中的 `TRAPFRAME` 的指针. `userret` 切换 `satp` 至用户页表。用户页表映射了 trampoline page 和 `TRAPFRAME` ，但不映射来自内核的其他页面。 trampoline page 在用户页表和内核页表映射在相同虚拟地址，所以 `uservec` 可以在改变 `satp` 后继续执行。

`userret` 将 trapframe 保存的用户`a0`拷贝到`sscratch`，以准备与`TRAPFRAME`交换。

## Code: Calling system calls

用户代码将exec的参数放在寄存器`a0`和`a1`，并将系统调用号码放在`a7`。系统调用编号匹配`syscalls`数组中的入口（一个函数指针表）。`ecall` trap 进内核然后执行 `uservec`，`usertrap`，`syscall`.

`syscall` 从`a7`检索系统调用编号，并将其当作`syscalls`的下标。

当系统调用返回时，`syscall` 在`p->trapframe->a0`记录返回值，

## Code: System call arguments



































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



