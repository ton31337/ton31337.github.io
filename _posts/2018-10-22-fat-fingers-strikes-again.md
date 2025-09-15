---
layout: post
title: DC meltdown by having fat-finger syndrome
categories:
- blog
---

In my [previous lighting blog post](https://donatas.net/blog/2018/01/24/exabgp-ipv6-prefix/), I described how it’s possible to null route the whole datacenter by having _fat-finger_ syndrome.

This is a second example of how to melt down your network by just mistyping a single character. Imagine what happens when you type:

```
neighbor 10.0.0.1 route-map lN in
...
route-map IN permit
  match ip address prefix-list default-only
```

Looking around with a _properly bad_ font you can’t catch anything abnormal. But in != ln.

In this case, you receive a full BGP table which will be inserted into the routing table and with a weak hardware you should guess what happens.

Neither Cisco nor Juniper does not handle this anyhow.

Again with my non-profit contribution marathon to [FRRouting](https://frrouting.org/) this raises [warning](https://github.com/FRRouting/frr/pull/3024/commits/1de27621531b996db577f67fb43483286571dbf2) if you mistype route-map name. It's much easier and faster to detect if you type those commands inside the terminal.
