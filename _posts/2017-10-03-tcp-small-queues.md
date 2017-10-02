---
layout: post
title: TCP Small Queues
categories:
- blog
---

I was always interested in receive (RX) side when handling the packets (dunno why), but when you dig into the congestion avoidance situations you MUST pay more attention to sending side (TX).

I read about BBR (the quite new congestion algorithm) invented by Google and there are a couple of interesting moving parts involved like TSO, TSQ, pacing, fair queuing before proceeding to NIC. All of those are well known but TSQ was untouched for me, though. So I decided to take a deeper look about TSQ and how it works internally.

>When sockets send and receive packets, they must be properly charged for the amount of system memory consumed by that packet. BSD sockets have the concept of send and receive buffers, which are used to keep track of this and make sure sockets stay within their limits.

Allocated memory constraints for the socket are set by using `sk->sk_sndbuf`/`sk->sk_rcvbuf`. This means you cannot buffer more data than socket allows. You send the packet to the socket and allocated memory which resides in `skb->truesize` + `struct sk_buff` overhead is accumulated using `atomic_add(skb->truesize, &sk->sk_wmem_alloc);` and vice versa `atomic_sub(skb->truesize - 1, &sk->sk_wmem_alloc);` substracted from `sk_wmem_allow` accordingly.

Looks very simple and nice. TSQ just works together with TSO as an additional layer to make absorption as little as needed. In short, it sits between TSO and qdisk+device queues to make sure latencies are much lower and to avoid bufferbloat circumstances. Without TSQ, TSO (basically allows putting more than 66560 bytes into a single packet) could grow the packets (1M or more), which could lead to high latencies.

While I didn't find anything useful about tuning or so for TSQ, I decided to look further. The best way to learn something is as always - look at the code.

```
% git log --pretty=oneline | grep -i 'small queue'
9b462d02d6dd671a9ebdc45caed6fe98a53c0ebe tcp: TCP Small Queues and strange attractors
b2532eb9abd88384aa586169b54a3e53574f29f8 tcp: fix ooo_okay setting vs Small Queues
46d3ceabd8d98ed0ad10f20c595ca784e34786c5 tcp: TCP Small Queues
d87c771f47460f7ca943942d508f2b9bd223a7e0 iwlegacy: small queue initializations cleanup

% git show 46d3ceabd8d98ed0ad10f20c595ca784e34786c5
...
```

### Enough shit - show me the numbers
```
~# stap -e 'global tsq; probe kernel.function("tcp_write_xmit") { tsq <<< $sk->sk_wmem_alloc->counter; } probe timer.s(10) { print(@hist_linear(tsq, 0, 262144, 65536)); exit(); }'
 value |-------------------------------------------------- count
     0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  851079
 65536 |                                                      196
131072 |                                                       37
196608 |                                                       80
262144 |                                                       33

~# sysctl -w net.ipv4.tcp_limit_output_bytes=65536 # 4x smaller than default

 value |-------------------------------------------------- count
     0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  789792
 65536 |                                                       88
131072 |                                                        0
196608 |                                                        0
```

This proves that TSQ really works as expected. TSO no more adds huge packets in flight thus reducing latencies.

#### So?

Does it require any further tuning? I don't think so unless you really like do (un)tune something. One needs specialized deep knowledge — it's best to have Eric Dumazet available ;-)
