---
title: '6.S081 Lab: Xv6 and Unix utilities'
date: 2022-09-11 22:21:44
tags:
- 6.S081
- lab
---

链接：[点击此处](https://pdos.csail.mit.edu/6.828/2020/labs/util.html)

## Boot xv6

安装在了wls上

> You can run make grade to test your solutions with the grading program.

## sleep  ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

> Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file `user/sleep.c`.

直接 Use the system call sleep 即可

## pingpong ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

> Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file `user/pingpong.c`.

很简单

## primes ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file `user/primes.c`.

[this page](http://swtch.com/~rsc/thread/) 很有意思

 ![img](https://swtch.com/~rsc/thread/sieve.gif) 

利用递归思想和`fork()`和`pipe()`进行实现

## find ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file `user/find.c`. 

魔改 `ls.c`

## xargs ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

利用 `|` 传输的数据直接用read读就好

![1662954828683](/6-S081-Lab-Xv6-and-Unix-utilities.assets/1662954828683.png)