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



## Print a page table

## A kernel page table per process

> xv6 有一个单独的 kernel page table，并且它是直接映射到物理地址中的。
>
> xv6 还为每个进程的 user address space 维护了单独的页表，只包含进程的 user memory 的映射。因为 kernel page table 不包含这些映射，user address 在 kernel 中是无效的。所以当 kernel 想要使用 system call 传递的 user pointer 时，kernel 首先要将这个指针转换成物理地址。
> 
> 本节和下节的目标是允许 kernel 直接使用 user pointer

本节任务是需要修改 kernel ，所以每个进程在 kernel 执行时 会使用这个进程自己的 kernel 的拷贝。

1. 修改 `struct proc`，为每个进程维护一个 `kernel page`
2. 修改 `scheduler` 使得在切换进程时可以切换 kernel page

所以  each per-process kernel page table should be identical to the existing global kernel page table
