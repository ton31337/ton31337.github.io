---
layout: post
title: Traffic steering using GeoDNS and IPv6
categories:
- blog
---

Using DNS-based load balancing doesn't save you from the failure. DNS server doesn't know if the backend is up or down. It just responds without carrying about the state of the backend. Eventually, your connections will timeout and/or be re-established to another instance (if TTL is small enough, e.g.: 30 seconds).

This could be improved using GeoDNS. GeoDNS server would respond with the target's address depending on the resolver's (or EDNS - client) IP.
Next level would be to introduce some CDN to cache static content and balance your traffic to arbitrary nodes using some defined rules.

More advanced option to balance the traffic is using anycast network. The main drawback using anycast is that this method is not always friendly for the end user and is expensive for the company. Expensive is because of maintenance, monitoring and $. You should allocate the whole _/24_ block for it (yeah, this is the smallest prefix length over global BGP table). So if you are planning to use only a few addresses from the whole /24 block - that's not the way to go.

One more interesting approach is using CDN in front and anycast at the backends. In this case, CDN provider will load balance your traffic using anycast address, but again you are not sure if your content is served from the right region.
To improve this setup we can install GeoDNS server and ask CDN's resolver to query CNAME record instead of A/AAAA. GeoDNS, in this case, will respond with appropriate backend's address according to resolver's (or EDNS) source IP.

>As I mentioned above IPv4 anycast is an expensive solution, but not for an old good friend - IPv6!

In this blog post, I would like to explain one more interesting approach on how to handle failovers gracefully. With IPv6 things change. As drawn in the diagram you should see that every PoP has two prefixes announced.

![ipv6-geodns-failover](/images/ipv6-geodns-failover.png)

One global anycast plus region allocated prefix. Both are overlapping prefixes which allow having smooth failover if one region goes down completely. For instance, your GeoDNS server responds to CNAME record with IP _2A02:4780:C1::1_ for the CDN's resolver and at that moment this region is down. New connections will be redirected to the shortest AS-PATH PoP because of global anycast overlapped network.

This is not for free of course. You should deploy this setup if you care about the infrastructure - it allows you to turn off the whole PoP (or DC) for testing (or Chaos engineering) purposes. Everything you are not testing is breaking.

>Chaos engineering will allow you to personally meet all of your colleagues within a short time – whether you want to or not!
