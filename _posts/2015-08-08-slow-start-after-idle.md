---
layout: post
title: TCP slow start, is it good or not?
categories:
- blog
---

## What are we talking about?

#### tcp_slow_start_after_idle

If set, provide RFC2861 behavior and time out the congestion window after an idle period. An idle period is defined at the current RTO. If unset, the congestion window will not be timed out after an idle period. Default: 1

#### tcp_no_metrics_save

By default, TCP saves various connection metrics in the route cache when the connection closes, so that connections established in the near future can use these to set initial conditions. Usually, this increases overall performance, but may sometimes cause performance degradation. If set, TCP will not cache metrics on closing connections.

## Tests (sent congestion window size)

#### tcp_slow_start_after_idle=0 and tcp_no_metrics_save=0

```
count: 16531, min: 5, max: 383, avg: 31
```

#### tcp_slow_start_after_idle=1 and tcp_no_metrics_save=0

```
count: 15250, min: 5, max: 383, avg: 10
```

#### tcp_slow_start_after_idle=0 and tcp_no_metrics_save=1

```
count: 15327, min: 5, max: 383, avg: 24
```

#### tcp_slow_start_after_idle=1 and tcp_no_metrics_save=1

```
count: 15862, min: 5, max: 383, avg: 10
```

You might wonder how are these numbers obtained? 
```
%{
#include <linux/tcp.h>
%}

global cwnd;
global counter;
global graph = 0;

function get_cwnd:long(sk:long)
%{
        struct tcp_sock *tp = tcp_sk((struct sock *)STAP_ARG_sk);
        if (tp->snd_cwnd)
                THIS->__retvalue = tp->snd_cwnd;
%}

probe kernel.function("tcp_ack").return
{
        cwnd <<< get_cwnd($sk);
        printf("%d\n", get_cwnd($sk));
        if (graph)
                if (counter++ > 500)
                        exit();
}

probe timer.s(30)
{
        printf("count: %d, min: %d, max: %d, avg: %d\n",
                @count(cwnd), @min(cwnd), @max(cwnd), @avg(cwnd));
        exit();
}
```

## Results

![](/images/tcp_slow_start.png)
Looks like the first case has the best performance, while the second has the worst performance. Even if you have `tcp_slow_start_after_idle` disabled, enabling `tcp_no_metrics_save` will win some performance. 
