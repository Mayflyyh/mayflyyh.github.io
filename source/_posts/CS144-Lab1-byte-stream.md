---
title: CS144 Lab1 byte stream
date: 2022-09-09 15:35:56
categories:
- CS144
tags:
- lab
- network
---

# CS144 Lab Checkpoint 1

##  Overview

Over the next four weeks, you’ll implement TCP, to provide the byte-stream abstraction between a pair of computers separated by an unreliable datagram network. 

<!-- more -->

![1662711801508](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/1662711801508.png)

Figure 1: The arrangement of modules and dataflow in your TCP implementation. The ByteStream was Lab. The job of TCP is to convey two **ByteStreams*** (one in each direction) over an unreliable datagram network, so that bytes written to the socket on one side of the connection emerge as bytes that can be read at the peer, and vice versa. Lab 1 is the **StreamReassembler***, and in Labs 2, 3, and 4 you’ll implement the **TCPReceiver**, **TCPSender**, and then the **TCPConnection** to tie it all together. 

---

In Lab 1, you’ll implement a stream reassembler—a module that stitches small pieces of the byte stream (known as substrings, or segments) back into a contiguous stream of bytes in the correct sequence.  

##  Getting started 

The TCP sender is dividing its byte stream up into short segments (substrings no more than about 1,460 bytes apiece) so that they each fit inside a datagram. But the network might reorder these datagrams, or drop them, or deliver them more than once. The receiver must reassemble the segments into the contiguous stream of bytes that they started out as.  



熟悉了字符串的使用（）

挺多细节很有意思，适合出成大模拟题！

![1662741049597](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/1662741049597.png)

第14个点慢应该是网络原因吧（）