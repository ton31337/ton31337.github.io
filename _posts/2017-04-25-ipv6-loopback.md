---
layout: post
title: Handling loopback interface in IPv6 stack
categories:
- blog
---

I read a few months ago or even more, that IPv6 stack is handling loopback differently than IPv4. In other words, it allows sources with `::1` as the real candidate from outside.

I checked kernel 4.8 sources and it has such a code for handling this only for the destination address. Basically, it checks if the destination address isn't loopback.

```
int ipv6_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
    ...
        if (!(dev->flags & IFF_LOOPBACK) &&
        ipv6_addr_loopback(&hdr->daddr))
            goto err;
    ...
}
```

while IPv4 stack has:

```
static int ip_route_input_slow(struct sk_buff *skb, __be32 daddr, __be32 saddr,
                               u8 tos, struct net_device *dev)
{
    ...
        if (ipv4_is_loopback(daddr)) {
                if (!IN_DEV_NET_ROUTE_LOCALNET(in_dev, net))
                        goto martian_destination;
        } else if (ipv4_is_loopback(saddr)) {
                if (!IN_DEV_NET_ROUTE_LOCALNET(in_dev, net))
                        goto martian_source;
        }
    ...
}
```

According to this, I'm able to send packets from outside (maybe local network would be more accurate regarding other reverse path filterings) interface with source as "originated from ::1". Looks promising ;-)

I'm not sure if it would be enough to introduce additional check for source address using the same way as with destination like (assuming that I don't have any free time for kinda hobby contributions):

```
ipv6_addr_loopback(&hdr->daddr) || ipv6_addr_loopback(&hdr->saddr)
```

But in general, it looks it could be solved trivially.

The current solution for this sort of "bug" is to filter out those abnormal packets using netfilter:

```
ip6tables -A INPUT -s ::1 -d ::1 -i lo -j ACCEPT
ip6tables -A INPUT -s ::1 -j DROP
```

#### Conclusion
* I'm still surprised - why it's still not fixed, while it's really not a big deal?;
* There is no synchronization between IPv4 and IPv6 stacks in Linux kernel.
