---
title: CS144 Lab2 TCP receiver
date: 2022-09-10 18:12:34
categories:
- CS144
tags:
- lab
- network
---

# CS144 Lab 2: The TCP Receiver

##  Overview

In Lab 2, you will implement the TCPReceiver, the part of a TCP implementation that handles the incoming byte stream. The TCPReceiver translates between incoming TCP segments (the payloads of datagrams carried over the Internet) and the incoming byte stream. 

<!-- more -->

[Read PDF](https://cs144.github.io/assignments/lab2.pdf)

In addition to writing to the incoming stream, the TCPReceiver is responsible for telling the sender two things: 

1. the index of the “first unassembled” byte, which is called the “acknowledgment number” or “ackno.” This is the first byte that the receiver needs from the sender. 
2. the distance between the “first unassembled” index and the “first unacceptable” index. This is called the “window size”.  

Together, the ackno and window size describe describes the receiver’s window: a range of indexes that the TCP sender is allowed to send. Using the window, the receiver can control the flow of incoming data, making the sender limit how much it sends until the receiver is ready for more. We sometimes refer to the ackno as the “left edge” of the window (smallest index the TCPReceiver is interested in), and the ackno + window size as the “right edge” (just beyond the largest index the TCPReceiver is interested in). 



## 3. Lab 2: The TCP Receiver

This week, you’ll implement the “receiver” part of TCP, responsible for receiving TCP segments (the actual datagram payloads), reassembling the byte stream (including its ending, when that occurs), and determining that signals that should be sent back to the sender for acknowledgment and flow control. 

###  3.1 Translating between 64-bit indexes and 32-bit seqnos

As a warmup, we’ll need to implement TCP’s way of representing indexes. Last week you created a StreamReassembler that reassembles substrings where each individual byte has a 64-bit stream index, with the first byte in the stream always having index zero. A 64-bit index is big enough that we can treat it as never overflowing.1 In the TCP headers, however, space is precious, and each byte’s index in the stream is represented not with a 64-bit index but with a 32-bit “sequence number,” or “seqno.” This adds three complexities: 

1. Your implementation needs to plan for 32-bit integers to wrap around. 
2. TCP sequence numbers start at a random value 
3. The logical beginning and ending each occupy one sequence number 

1. 

> `WrappingInt32 wrap(uint64 t n, WrappingInt32 isn)` 
>
> Convert `absolute seqno` → `seqno`. Given an absolue sequence number (n) and an Initial Sequence Number ($isn$), produce the (relative) sequence number for $n$.

$n+isn$ 即可

```c
WrappingInt32 wrap(uint64_t n, WrappingInt32 isn) {
    DUMMY_CODE(n, isn);
    uint64_t isn64 = isn.raw_value();
    uint32_t res = n + isn64 ; 
    return WrappingInt32{res};
}
```

2. 

> `uint64 t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64 t checkpoint)`
>
> Convert `seqno` → `absolute seqno`. Given a sequence number ($n$), the Initial Sequence Number ($isn$), and an absolute checkpoint sequence number, compute the absolute sequence number that corresponds to n that is closest to the checkpoint. 

先计算出 $n$ 对于 $checkpoint$ 在`wrap`下的偏移量，然后再加回  $checkpoint$，若小于 $0$ 则加一个$2^{32}$ 即可（这种情况会发生在 $(int)n<0$ 时）。

```c
uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint) {
    DUMMY_CODE(n, isn, checkpoint);
    long long diff = n - wrap(checkpoint,isn) + checkpoint;
    if(diff<0) diff += 1ll<<32;
    return static_cast<uint64_t> (diff);
}
```

###  3.2 Implementing the TCP receiver 

Congratulations on getting the wrapping and unwrapping logic right! We’d shake your hand if we could. In the rest of this lab, you’ll be implementing the TCPReceiver. It will 

1. receive segments from its peer
2. reassemble the ByteStream using your StreamReassembler 
3. calculate the acknowledgment number (ackno) and the window size. The ackno and window size will eventually be transmitted back to the peer in an outgoing segment.  

First, please review the format of a TCP segment. This is the message that the two endpoints send each other; it is the payload of the lower-level datagrams. The non-grayed-out fields represent the information that’s of interest in this lab: 

- the sequence number

- the payload

- the SYN and FIN flags. 

These are the fields that are written by the sender, and read and acted on by the receiver. 



![1663167576612](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/1663167576612.png)
