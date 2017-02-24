---
layout: post
title: Unexpected network latency with IPv6
categories:
- blog
---

Interesting things are happening in IPv6 world and I like them. This is especialy interesting. We have up to 512 IPv6 addresses per node (all of them except management address are announced by BGP and set under loopback interface). Take this example with ping:

```
# ping6 2a02:4780:bad:c0de::d
PING 2a02:4780:bad:c0de::d(2a02:4780:bad:c0de::d) 56 data bytes
64 bytes from 2a02:4780:bad:c0de::d: icmp_seq=1 ttl=63 time=6.39 ms
64 bytes from 2a02:4780:bad:c0de::d: icmp_seq=2 ttl=63 time=7.59 ms

# ping 153.92.0.72
PING 153.92.0.72 (153.92.0.72) 56(84) bytes of data.
64 bytes from 153.92.0.72: icmp_seq=1 ttl=63 time=0.316 ms
64 bytes from 153.92.0.72: icmp_seq=2 ttl=63 time=0.336 ms
```

You should notice that `ping6` is having abnormal latency in local network (it's a 10GE connection). Looking at perf events, we see `ipv6_addr_label()` is a good candidate to ask.

```
Samples: 28  of event 'cycles', Event count (approx.): 22131542
  Children      Self  Command  Shared Object      Symbol
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] entry_SYSCALL_64_fastpath
+  100.00%     0.00%  ping6    [unknown]          [.] 0x0df0ad0b8047022a
+  100.00%     0.00%  ping6    libc-2.17.so       [.] __sendto_nocancel
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] sys_sendto
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] SYSC_sendto
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] sock_sendmsg
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] inet_sendmsg
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] rawv6_sendmsg
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] ip6_dst_lookup_flow
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] ip6_dst_lookup_tail
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] ip6_route_get_saddr
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] ipv6_dev_get_saddr
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] __ipv6_dev_get_saddr
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] ipv6_get_saddr_eval
+  100.00%     0.00%  ping6    [kernel.kallsyms]  [k] ipv6_addr_label
+  100.00%   100.00%  ping6    [kernel.kallsyms]  [k] __ipv6_addr_label
+    0.00%     0.00%  ping6    [kernel.kallsyms]  [k] schedule
```

Let's count how often `ipv6_addr_label()` is called:

```
# ./kernel/funccount -i 1 'ipv6_addr_label'
Tracing "ipv6_addr_label"... Ctrl-C to end.

FUNC                              COUNT
ipv6_addr_label                   11586
```

Kernel calls this way to often.. And this happens because we have huge amount of IPv6 addresses on a single node and kernel tries to pick the best candidate for outgoing connection. Rule 6 in source address selection ("prefer matching label") does a lookup in a table of prefixes to find a label for the destination address. It then does the same lookup for each of the candidate source addresses, and prefers those that have the same label as the destination address.

Let's figure out which label we are striking here:

```
#!/usr/bin/env python

from bcc import BPF

bcc_prog = """
#include <uapi/linux/ptrace.h>
#include <net/sock.h>
#include <bcc/proto.h>
int kretprobe__ipv6_addr_label(struct pt_regs *ctx,
                               struct net *net,
                               const struct in6_addr *addr,
                               int type,
                               int ifindex)
{
        int ret = PT_REGS_RC(ctx);
        bpf_trace_printk("ifindex: %d, label: %d\\n", ifindex, ret);
        return 0;
}
"""

b = BPF(text=bcc_prog)
b.trace_print()

"""
           ping6-5707  [000] d... 167873.402854: : ifindex: -1, label: 6
           ping6-5707  [000] d... 167873.404414: : ifindex: -1, label: 6
           ping6-5707  [000] d... 167874.406325: : ifindex: -1, label: 6
           ping6-5707  [000] d... 167875.408850: : ifindex: -1, label: 6
"""
```

What is under label 6?

```
# ip addrlabel | grep 'label 6'
prefix 2001::/32 label 6
```

Okay, looks like general things, let's try to avoid rule six at all and give precedence by manually picking the interface.

```
# sysctl -w net.ipv6.conf.ens1f0.use_oif_addrs_only=1
# ping6 2a02:4780:bad:c0de::2
PING 2a02:4780:bad:c0de::2(2a02:4780:bad:c0de::2) 56 data bytes
64 bytes from 2a02:4780:bad:c0de::2: icmp_seq=1 ttl=63 time=0.209 ms
64 bytes from 2a02:4780:bad:c0de::2: icmp_seq=2 ttl=63 time=0.307 ms
64 bytes from 2a02:4780:bad:c0de::2: icmp_seq=3 ttl=63 time=0.343 ms

# ./kernel/funccount -i 1 'ipv6_addr_label'
Tracing "ipv6_addr_label"... Ctrl-C to end.

FUNC                              COUNT
ipv6_addr_label                    2638
```

Great success \o/
