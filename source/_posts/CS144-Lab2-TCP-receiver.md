---
title: CS144 Lab2 TCP receiver
date: 2022-09-10 18:12:34
categories:
- CS144
tags:
- lab
- network
---

# CS144 Lab Checkpoint 2

##  Overview

In Lab 2, you will implement the TCPReceiver, the part of a TCP implementation that handles the incoming byte stream. The TCPReceiver translates between incoming TCP segments (the payloads of datagrams carried over the Internet) and the incoming byte stream. 

<!-- more -->

In addition to writing to the incoming stream, the TCPReceiver is responsible for telling the sender two things: 

1. the index of the “first unassembled” byte, which is called the “acknowledgment number” or “ackno.” This is the first byte that the receiver needs from the sender. 
2. the distance between the “first unassembled” index and the “first unacceptable” index. This is called the “window size”.  

Together, the ackno and window size describe describes the receiver’s window: a range of indexes that the TCP sender is allowed to send. Using the window, the receiver can control the flow of incoming data, making the sender limit how much it sends until the receiver is ready for more. We sometimes refer to the ackno as the “left edge” of the window (smallest index the TCPReceiver is interested in), and the ackno + window size as the “right edge” (just beyond the largest index the TCPReceiver is interested in). 