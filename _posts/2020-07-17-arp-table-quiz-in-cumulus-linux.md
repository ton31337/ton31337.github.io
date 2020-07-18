---
layout: post
title: ARP table quiz in Cumulus Linux
categories:
- blog
---

The weird situation when you notice that IPv4 get a packet loss, while IPv6 works as expected for the same physical link:

```
--- 2a02:4780:face:f00d::1 ping statistics ---
1368 packets transmitted, 1368 received, 0% packet loss, time 1367006ms
rtt min/avg/max/mdev = 0.217/0.285/0.667/0.045 ms
root@sg-nme-leaf1:~#

--- 10.0.31.1 ping statistics ---
1366 packets transmitted, 1348 received, 1% packet loss, time 1365005ms
rtt min/avg/max/mdev = 0.147/0.227/0.874/0.064 ms
root@sg-nme-leaf1:~#
```

There is a lot of what to check, but this is what I did solving this issue.

First I checked [ptmd](https://docs.cumulusnetworks.com/cumulus-linux-41/Layer-1-and-Switch-Ports/Prescriptive-Topology-Manager-PTM/) status which surprised me more. BFD status was failed, but
BGP session was UP. That's because we use a single IPv6 session for both IPv4 and IPv6.

10.0.31.1 is down while 2a02:4780:face:f00d::1 is up. Why? It's for the same reason (packet loss).

Checking for drops/errors with `ethtool -S swp48 | grep -iE "drop|err|disc"` gave nothing except that there is a usual drops count which is normal due ACL, bursty buffer congestions, etc.

If IPv6 works while IPv4 not, it seems related to the ARP table. I double-checked ARP table entries with `ip neigh | wc -l`. It was around 6k. Nothing special as well, just a well-worn node.

Unfortunately, Broadcom devices do not have native ASIC monitoring, which could provide me with the stats about the buffers, packets count, queue lengths, etc.

I thought I would inspect buffer congestions or so, but anyway. Continuing.

Running this command in a terminal I noticed host entries drop when packet loss happens:
```
watch -n 1 'date >> /tmp/host_count.log ; cat /cumulus/switchd/run/route_info/host/count_0 >> /tmp/host_count.log ; tail /tmp/host_count.log'

Fri Jul 17 08:14:06 UTC 2020
12702
Fri Jul 17 08:14:07 UTC 2020
9281
```

Like from 12k to 9k and varying. The maximum is 16k, but that's not an issue since it's not hitting nearly 16k.

`dmesg` is clear. If it would be garbage collection for a stale ARP entries it would be an error message in `dmesg` output, like: `kernel: Neighbour table overflow`.

Just in case I checked `net.ipv4.neigh.default.gc_thresh1` which was a default 128. Like I mentioned above current ARP entries were around 6k.

128 is the maximum number of ARP entries in the cache. Garbage collection won't be triggered if below 128. We have 6k. That's why I noticed a host entries drop when packet loss happens. I double-checked this few times and confirmed.

Changed to gc_thresh1 to 8192 (bigger than we have ARP entries) and the problem just gone.
