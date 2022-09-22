---
title: CS144 Lab3 TCP sender
date: 2022-09-21 22:34:03
categories:
- CS144
tags:
- lab
- network
---

The **TCPSender** is a tool that translates from an outgoing byte stream to segments that will become the payloads of unreliable datagrams.

<!-- more -->

Lab0: the abstraction of a flow-controlled byte stream (**ByteStream**).

Lab1,2 the tools that translate from segments carried in unreliable datagrams to an incoming byte stream: the **StreamReassembler** and **TCPReceiver**

This Lab3:

The **TCPSender** is a tool that translates from an outgoing byte stream to segments that will become the payloads of unreliable datagrams.

![image-20220921224147529](https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220921224147529.png)

## Lab 3: The TCP Sender

This week, you’ll implement the “sender” part of TCP, responsible for reading from a ByteStream (created and written to by some sender-side application), and turning the stream into a sequence of outgoing TCP segments. On the remote side, a TCP receiver transforms those segments (those that arrive—they might not all make it) back into the original byte stream, and sends acknowledgments and window advertisements back to the sender.

It will be your TCPSender’s responsibility to:

- Keep track of the receiver’s window (processing incoming acknos and window sizes)
- Fill the window when possible, by reading from the ByteStream, creating new TCP segments (including SYN and FIN flags if needed), and sending them. The sender should keep sending segments until either the window is full or the ByteStream is empty.
- Keep track of which segments have been sent but not yet acknowledged by the receiver— we call these “outstanding” segments
- Re-send outstanding segments if enough time passes since they were sent, and they haven’t been acknowledged yet

The basic principle is to send whatever the receiver will allow us to send (filling the window), and keep retransmitting until the receiver acknowledges each segment. This is called “automatic repeat request” (ARQ). The sender divides the byte stream up into segments and sends them, as much as the receiver’s window allows.

---

<img src="https://raw.githubusercontent.com/Mayflyyh/picrepo/main/image-20220922190530721.png" alt="image-20220922190530721" style="zoom:50%;" />

写完了，但欲哭无泪，犯了很多细节错误

```c++
//tcp_sender.hh

#ifndef SPONGE_LIBSPONGE_TCP_SENDER_HH
#define SPONGE_LIBSPONGE_TCP_SENDER_HH

#include "byte_stream.hh"
#include "tcp_config.hh"
#include "tcp_segment.hh"
#include "wrapping_integers.hh"

#include <cassert>
#include <functional>
#include <iostream>
#include <queue>
#include <set>

class TCPTimer {
  private:
    // 全局时间
    size_t _current_time;
    // TCPTimer启动时的时间
    size_t _start_time;
    // 是否处于RUNNING状态
    bool _RUNNING;

  public:
    TCPTimer();
    void start(bool restart);
    void stop();
    void updateTime(size_t _time);
    size_t getTimeGap();
};

//! \brief The "sender" part of a TCP implementation.

//! Accepts a ByteStream, divides it up into segments and sends the
//! segments, keeps track of which segments are still in-flight,
//! maintains the Retransmission Timer, and retransmits in-flight
//! segments if the retransmission timer expires.
class TCPSender {
  private:
    //! our initial sequence number, the number for our SYN.
    WrappingInt32 _isn;

    //! outbound queue of segments that the TCPSender wants sent
    std::queue<TCPSegment> _segments_out{};
    std::queue<TCPSegment> _outstanding_segments{};

    //! retransmission timer for the connection
    unsigned int _initial_retransmission_timeout;
    unsigned int _current_retransmission_timeout;

    //! outgoing stream of bytes that have not yet been sent
    ByteStream _stream;

    //! the (absolute) sequence number for the next byte to be sent
    uint64_t _next_seqno{0};
	
    //! 保存最新的 ackno
    uint64_t _ackno{0};
	
    //! 保存最新的window_size
    uint64_t _window_size;

    // 时间记录器
    TCPTimer _timer;
    
    //! 记录已发送但还没有被确认的字节数
    uint64_t _bytes_in_flight{0};
    
    //! 记录重发次数
    uint64_t _consecutive_retransmissions{0};
    bool _end{false};

  public:
    //! Initialize a TCPSender
    TCPSender(const size_t capacity = TCPConfig::DEFAULT_CAPACITY,
              const uint16_t retx_timeout = TCPConfig::TIMEOUT_DFLT,
              const std::optional<WrappingInt32> fixed_isn = {});

    //! \name "Input" interface for the writer
    //!@{
    ByteStream &stream_in() { return _stream; }
    const ByteStream &stream_in() const { return _stream; }
    //!@}

    //! \name Methods that can cause the TCPSender to send a segment
    //!@{

    //! \brief A new acknowledgment was received
    void ack_received(const WrappingInt32 ackno, const uint16_t window_size);

    //! \brief Generate an empty-payload segment (useful for creating empty ACK segments)
    void send_empty_segment();

    //! \brief create and send segments to fill as much of the window as possible
    void fill_window();

    //! \brief Notifies the TCPSender of the passage of time
    void tick(const size_t ms_since_last_tick);
    //!@}

    //! \name Accessors
    //!@{

    //! \brief How many sequence numbers are occupied by segments sent but not yet acknowledged?
    //! \note count is in "sequence space," i.e. SYN and FIN each count for one byte
    //! (see TCPSegment::length_in_sequence_space())
    size_t bytes_in_flight() const;

    //! \brief Number of consecutive retransmissions that have occurred in a row
    unsigned int consecutive_retransmissions() const;

    //! \brief TCPSegments that the TCPSender has enqueued for transmission.
    //! \note These must be dequeued and sent by the TCPConnection,
    //! which will need to fill in the fields that are set by the TCPReceiver
    //! (ackno and window size) before sending.
    std::queue<TCPSegment> &segments_out() { return _segments_out; }
    //!@}

    //! \name What is the next sequence number? (used for testing)
    //!@{

    //! \brief absolute seqno for the next byte to be sent
    uint64_t next_seqno_absolute() const { return _next_seqno; }

    //! \brief relative seqno for the next byte to be sent
    WrappingInt32 next_seqno() const { return wrap(_next_seqno, _isn); }
    //!@}
};

#endif  // SPONGE_LIBSPONGE_TCP_SENDER_HH
```

以上是头文件

### Before start

- What should my TCPSender assume as the receiver’s window size before I’ve gotten an ACK from the receiver? 

  One byte.

- Wait, how do I both “send” a segment and also keep track of that same segment as being outstanding, so I know what to retransmit later? Don’t I have to make a copy of each segment then? Is that wasteful? 

  When you send a segment that contains data, you’ll probably want to push it on to the segments out queue and also keep a copy of it internally in a data structure that lets you keep track of outstanding segments for possible retransmission. This turns out not to be very wasteful because the segment’s payload is stored as a reference-counted read-only string (a Buffer object). So don’t worry about it—it’s not actually copying the payload data.

这里也说到了如果会通过引用计数的 Buffer 来优化空间，不会造成空间浪费。

### TCPTimer

```c++
//tcp_sender.cc
TCPTimer::TCPTimer() : _current_time(0), _start_time(0), _RUNNING(0) {}

void TCPTimer::start(bool restart) {
    //restart 是否是重新启动？
    if (restart || _RUNNING == false) {
        _RUNNING = true;
        _start_time = _current_time;
    }
}

void TCPTimer::stop() { _RUNNING = false; }

void TCPTimer::updateTime(size_t _time) { _current_time += _time; }

size_t TCPTimer::getTimeGap() { return _current_time - _start_time; }
```

`TCPTimer` 类，用于更新当前时间，以及记录开始计时时间和停止时间，以及获取当前时间与现在时间的差（会与 retransmission timeout (RTO) 进行比较）

- When to start ?

	>How does the TCPSender know if a segment was lost?
	>
    >4. Every time a segment containing data (nonzero length in sequence space) is sent (whether it’s the first time or a retransmission), if the timer is not running, **start **it running so that it will expire after RTO milliseconds (for the current value of RTO).
    >
    >   By “expire,” we mean that the time will run out a certain number of milliseconds in the future.

也就是说当发送一段 sequence space 不为0的 tcpsegment 时，如果 timer 不处于 running状态，那么此时就开始计时。

- When to stop ? 


  >5. When all outstanding data has been acknowledged, **stop **the retransmission timer

### fill_window

> The TCPSender is asked to fill the window: it reads from its input ByteStream and sends as many bytes as possible in the form of TCPSegments, as long as there are new bytes to be read and space available in the window. You’ll want to make sure that every TCPSegment you send fits fully inside the receiver’s window. Make each individual TCPSegment as big as possible, but no bigger than the value given by TCPConfig::MAX PAYLOAD SIZE (1452 bytes). You can use the TCPSegment::length in sequence space() method to count the total number of sequence numbers occupied by a segment. Remember that the SYN and FIN flags also occupy a sequence number each, which means that they occupy space in the window.

```c++
void TCPSender::fill_window() {
    if (_end) {
        return;
    }
    size_t size = _ackno + (_window_size ? _window_size : 1) - _next_seqno;

    while (size > 0 && !_end) {
        TCPSegment message;
        TCPHeader &header = message.header();
        Buffer &payload = message.payload();
        header.seqno = wrap(_next_seqno, _isn);

        if (_next_seqno == 0) {
            header.syn = true;
            size -= 1;
        }

        payload = _stream.read(min(size, TCPConfig::MAX_PAYLOAD_SIZE));
        size -= payload.size();

        if (_stream.eof() && size > 0) {
            header.fin = true;
            size -= 1;
            _end = true;
        }

        if (message.length_in_sequence_space() == 0) {
            break;
        }

        _bytes_in_flight += message.length_in_sequence_space();
        _next_seqno = _next_seqno + message.length_in_sequence_space();

        _segments_out.push(message);
        _outstanding_segments.push(message);

        _timer.start(false);
    }
}
```

L5 :

> What should I do if the window size is zero? 
>
> If the receiver has announced a window size of zero, the fill window method should act like the window size is one. The sender might end up sending a single byte that gets rejected (and not acknowledged) by the receiver, but this can also provoke the receiver into sending a new acknowledgment segment where it reveals that more space has opened up in its window. Without this, the sender would never learn that it was allowed to start sending again.

就是说在 fill window 时，如果 window size 是 0，那么就将 window size 视为 1 ，这样才能保持 sender 继续向 receiver 发送信息，如果视为 0 那么就不能向 receiver 发送任何信息了。

然后定义 size 为 当前 window 剩余的 size 大小，window 的范围是 $[ackno,ackno+window size]$ 

> void ack received()
>
> 2. A segment is received from the receiver, conveying the new left (= ackno) and right (= ackno + window size) edges of the window

这里要明确 `sequence number` 这个概念，详见 Lab2 的 

> 3. The logical beginning and ending each occupy one sequence number: In addition to ensuring the receipt of all bytes of data, TCP makes sure that the beginning and ending of the stream are received reliably. Thus, in TCP the **SYN** (beginning-ofstream) and **FIN** (end-of-stream) control flags are assigned sequence numbers. Each of these occupies one sequence number. (The sequence number occupied by the SYN flag is the ISN.) **Each byte of data in the stream also occupies one sequence number**. Keep in mind that SYN and FIN aren’t part of the stream itself and aren’t “bytes”—they represent the beginning and ending of the byte stream itself.

也就是说，只有 SYN ，FIN，和 TCPSegment 中的 payload 在计算 sequence number 被考虑到，占用 sequence space

L18: 需要考虑 `_stream` 以及 `_stream.read(int)` 的一些特性

L27: 不需要发送 sequence space 为空的 TCPSegment



### ack_received

> A segment is received from the receiver, conveying the new left (= ackno) and right (= ackno + window size) edges of the window. The TCPSender should look through its collection of outstanding segments and remove any that have now been fully acknowledged (the ackno is greater than all of the sequence numbers in the segment). The TCPSender should fill the window again if new space has opened up.

```c++
void TCPSender::ack_received(const WrappingInt32 ackno, const uint16_t window_size) {
    DUMMY_CODE(ackno, window_size);
    uint64_t _new_ackno = unwrap(ackno, _isn, _ackno);

    bool isnew_ack = true;

    if (_new_ackno > _next_seqno)
        return;

    _window_size = window_size;
    if (_new_ackno <= _ackno)
        isnew_ack = false;

    if (isnew_ack) {
        _ackno = _new_ackno;
        while (!_outstanding_segments.empty() && ackno != (_outstanding_segments.front().header().seqno +
                                                           _outstanding_segments.front().length_in_sequence_space())) {
            if (!_outstanding_segments.empty() &&
                _new_ackno < unwrap(_outstanding_segments.front().header().seqno, _isn, _ackno) +
                                 static_cast<uint64_t>(_outstanding_segments.front().length_in_sequence_space())) {
                return;
            }
            _ackno = _new_ackno;
            _bytes_in_flight -= _outstanding_segments.front().length_in_sequence_space();
            _outstanding_segments.pop();
        }
        if (!_outstanding_segments.empty() && ackno == (_outstanding_segments.front().header().seqno +
                                                        _outstanding_segments.front().length_in_sequence_space())) {
            if (!_outstanding_segments.empty() &&
                _new_ackno < unwrap(_outstanding_segments.front().header().seqno, _isn, _ackno) +
                                 static_cast<uint64_t>(_outstanding_segments.front().length_in_sequence_space())) {
                return;
            }
            _ackno = _new_ackno;
            _bytes_in_flight -= _outstanding_segments.front().length_in_sequence_space();
            _outstanding_segments.pop();
        }

        _current_retransmission_timeout = _initial_retransmission_timeout;
        _timer.stop();

        if (!_outstanding_segments.empty()) {
            _timer.start(true);
            _current_retransmission_timeout = _initial_retransmission_timeout;
        }
        _consecutive_retransmissions = 0;
    }
}
```

> When the receiver gives the sender an ackno that acknowledges the successful receipt of new data (the ackno reflects an absolute sequence number bigger than any previous ackno): 
>
> (a) Set the RTO back to its “initial value.” 
>
> (b) If the sender has any outstanding data, restart the retransmission timer so that it will expire after RTO milliseconds (for the current value of RTO). 
>
> (c) Reset the count of “consecutive retransmissions” back to zero.

当收到新的 ackno 时需要将 RTO 置回初始值，如果需要计数则重新开始计数，并且将连续重发次数置回0

这里细节比较多，我有一个问题是

> 真实世界中会出现返回的ack是从整段seq中间截断的情况吗？
> 比如发一个seq=1,"abcdefg"，返回ack=3这样。



### tick

> Time has passed — a certain number of milliseconds since the last time this method was called. The sender may need to retransmit an outstanding segment.

```c++
void TCPSender::tick(const size_t ms_since_last_tick) {
    DUMMY_CODE(ms_since_last_tick);
    _timer.updateTime(ms_since_last_tick);
    if (!_outstanding_segments.empty() && _current_retransmission_timeout <= _timer.getTimeGap()) {
        if (!_outstanding_segments.empty())
            _segments_out.push(_outstanding_segments.front());
        if (_window_size) {
            _consecutive_retransmissions += 1;
            _current_retransmission_timeout *= 2;
        }
        _timer.start(true);
    }
}
```

> 2. When the TCPSender is constructed, it’s given an argument that tells it the “initial value” of the retransmission timeout (RTO). The RTO is the number of milliseconds to wait before resending an outstanding TCP segment. The value of the RTO will change over time, but the “initial value” stays the same. The starter code saves the “initial value” of the RTO in a member variable called initial retransmission timeout.
>
> 6. If tick is called and the retransmission timer has expired: 
>
>    (a). Retransmit the earliest (lowest sequence number) segment that hasn’t been fully acknowledged by the TCP receiver. You’ll need to be storing the outstanding segments in some internal data structure that makes it possible to do this. 
>
>    (b). If the window size is nonzero: i. Keep track of the number of consecutive retransmissions, and increment it because you just retransmitted something. Your TCPConnection will use this information to decide if the connection is hopeless (too many consecutive retransmissions in a row) and needs to be aborted. ii. Double the value of RTO. This is called “exponential backoff”—it slows down retransmissions on lousy networks to avoid further gumming up the works. 
>
>    (c). Reset the retransmission timer and start it such that it expires after RTO milliseconds (taking into account that you may have just doubled the value of RTO!).

注意重发时不需要把重发的 TCPSegment 当作新的 TCPSegment 而重新追踪。

### send empty segment

> The TCPSender should generate and send a TCPSegment that has zero length in sequence space, and with the sequence number set correctly. This is useful if the owner (the TCPConnection that you’re going to implement next week) wants to send an empty ACK segment. 
>
> Note: a segment like this one, which occupies no sequence numbers, doesn’t need to be kept track of as “outstanding” and won’t ever be retransmitted.

```c++
void TCPSender::send_empty_segment() {
    TCPSegment message;
    TCPHeader &header = message.header();
    header.seqno = wrap(_next_seqno, _isn);
    _segments_out.push(message);
}
```

