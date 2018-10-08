---
layout: post
title: Filter out routes by type of blackhole in FRRouting
categories:
- blog
---

#### SYN

Basically, there are two purposes of using blackhole routes:
* To prevent from DDoS attacks
* To inject fake routes into the routing table with longer network mask (e.g.: for BGP advertisements)

The former can also be implemented using well-known `iptables`. But it always depends on what is better `iptables`, `iproute2` using blackhole, `nftables`, `eBPF`, `XDP` or finally `XDP offload on the hardware` when talking about _good enough_.

If you have thousands of rules in `iptables` then the kernel has a lot of work to do by touching every rule in the list. In other words, when you modify a single rule, `iptables` has to fetch the whole basket of rules from the kernel (of course using locking because it's a linked-list and synchronization between CPUs), do the change and put all rules back. So, avoid having a plethora of rules under `iptables` - otherwise it will take half a minute to reload.

I'm pretty sure that 99% of servers (if not using the route on the host) has a routing table only with a default route, which means that dropping packets to specific destinations is much cheaper than using `iptables`.

```
ip route add blackhole 192.168.0.1/32
ip -6 route add blackhole 2a02:4780:dead::beef/128
```
vs.
```
iptables -A INPUT -d 192.168.0.1/32 -j DROP
ip6tables -A INPUT -d 2a02:4780:dead::beef/128 -j DROP
```

The difference in processing the packets is that `blackhole` doesn't send any packets back to the source while `netfilter` generates ICMP responses.

Let's say you want to filter out blackhole routes from being advertised to BGP neighbors, what you can do? If it's `/32` or `/128` then it can be filtered using `prefix-list`, otherwise can't.

With my latest [patch](https://github.com/FRRouting/frr/pull/3102/commits/61ad901e57049995709bcffe1ffaee65a9927a00) to [FRRouting](https://frrouting.org/) it can be implemented just by matching the type of the next-hop like:

```
route-map bh deny 10
  match ip next-hop type blackhole
route-map bh deny 20
  match ipv6 next-hop type blackhole
```

#### FIN
