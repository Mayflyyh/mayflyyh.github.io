---
title: '6.S081 Lab: System call tracing'
date: 2022-09-13 22:36:36
categories:
- 6.S081
tags:
- OS
- lab
---

还好

<!-- more -->

## System call tracing

>  In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new `trace` system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The `trace` system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes. 

- Run make qemu and you will see that the compiler cannot compile `user/trace.c`, because the user-space stubs for the system call don't exist yet: add a prototype for the system call to `user/user.h`, a stub to `user/usys.pl`, and a syscall number to `kernel/syscall.h`. The Makefile invokes the perl script `user/usys.pl`, which produces `user/usys.S`, the actual system call stubs, which use the RISC-V `ecall` instruction to transition to the kernel. Once you fix the compilation issues, run trace 32 grep hello README; it will fail because you haven't implemented the system call in the kernel yet.

首先在 `user/user.h` 中 添加  prototype for system call ，

> A function **prototype** is a **declaration in the code that instructs the compiler about the data type of the function, arguments and parameter list.**

```c
int trace(int);
```

然后在`to user/usys.pl` 添加 stub 

> A method stub or simply stub in software development is **a piece of code used to stand in for some other programming functionality**. A stub may simulate the behavior of existing code (such as a procedure on a remote machine; such methods are often called mocks) or be a temporary substitute for yet-to-be-developed code.

```c
entry("trace");
```

最后在 `kernel/syscall.h`  添加 `syscall number`，按部就班模仿就好

```c
#define SYS_trace  22
```

- Add a `sys_trace()` function in `kernel/sysproc.c` that implements the new system call by remembering its argument in a new variable in the `proc` structure (see `kernel/proc.h`). The functions to retrieve system call arguments from user space are in `kernel/syscall.c`, and you can see examples of their use in `kernel/sysproc.c`.

在 `kernel/sysproc.c` 添加 `sys_trace()`

```c
uint64
sys_trace(void)
{
    int n;

    if(argint(0, &n) < 0)
      return -1;
    myproc()->mask = n;
    return 0;
}
```

在 ` kernel/proc.h ` 中保存进程状态的 `kernel/proc.h` 添加 `mask` 

- Modify `fork()` (see `kernel/proc.c`) to copy the trace mask from the parent to the child process.

然后每当 `fork()` 时，需要将父进程的 `mask` 复制给子进程的 `mask`

- Modify fork() (see kernel/proc.c) to copy the trace mask from the parent to the child process.

```c
np->mask = p->mask;
```

然后每当 ` kernel/syscall.c ` 中的 `syscall()` 被调用时，都会检索这个进程的`mask `，如果所调用的 syscall function 在 `mask` 中（位运算），那就要需要打印出这个进程的进程号和 syscall function 的函数名以及返回值。其中 syscall 的函数名需要自己建一个数组。

```c
if(((p->mask)>>num)&1)
        printf("%d: syscall %s -> %d\n",p->pid,syscallname[num] , p->trapframe->a0);
```



## Sysinfo

> - Add `$U/_sysinfotest` to UPROGS in Makefile
>
> - Run make qemu; `user/sysinfotest.c` will fail to compile. Add the system call sysinfo, following the same steps as in the previous assignment. To declare the prototype for sysinfo() `in user/user.h` you need predeclare the existence of `struct sysinfo`:
>
>   ```c
>   struct sysinfo;
>   int sysinfo(struct sysinfo *);
>   ```
>
>​	   Once you fix the compilation issues, run sysinfotest; it will fail because    you haven't implemented the system call in the kernel yet.

在`user/user.h`中添加`struct sysinfo`和`int sysinfo`的预定义即可

> - sysinfo needs to copy a `struct sysinfo` back to user space; see `sys_fstat()` (`kernel/sysfile.c`) and `filestat()` (`kernel/file.c`) for examples of how to do that using `copyout()`.

在kernel/sysproc.c中添加`sys_sysinfo()`函数，在`kernel/kalloc.c`中实现对未占用内存信息的收集，在`kernel/proc.c`中实现对进程数信息的收集

```c
//kernel/sysproc.c

uint64
sys_sysinfo(void)
{
    struct sysinfo info;
    uint64 addr;
    struct proc *p;

    if(argaddr(0,&addr)<0){
        return -1;
    }
    p = myproc();
    info.freemem = kfreesize();
    info.nproc = proccount();
    if(copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
      return -1;
    return 0;
}
```

copyout就是将kernel space的东西拷贝到user space

the amount of free memory 在 xv6 中就是查看有多少未被分配的页表

```c
//kernel/kalloc.c

uint64 
kfreesize(void)
{
    struct run *ptr ;
    int cnt;

    cnt = 0 ;
    ptr = kmem.freelist;
    while(ptr){
        cnt++;
        ptr = ptr->next;
    }

    return cnt * PGSIZE;
}
```

the number of processes 即为多少进行部署unused

```c
//kernel/proc.c
int
proccount(void){
  struct proc *p;
  int cnt = 0 ;

  for(p = proc; p < &proc[NPROC]; p++){
    if(p->state != UNUSED)
      cnt++;
  }

  return cnt;
}
```




![](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/1663128399812.png)



