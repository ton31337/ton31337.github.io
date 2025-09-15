---
layout: post
title: Traffic steering using GeoDNS and IPv6
categories:
- blog
---

Using DNS-based load balancing doesn't save you from the failure. DNS server doesn't know if the backend is up or down. It just responds without carrying about the state of the backend.

So if you have two or more IPs per record, DNS server will respond in a round-robin manner. Let’s say one backend of two is under maintenance (down) and another is alive. Eventually, all connections affected by first backend failures will be re-established to another instance (if TTL is small enough, e.g.: 30 seconds).

Typically TTL is set to one hour or so. Some resolvers override source TTL to cache DNS responses longer. If you use DNS as a balancing layer then small TTL must be used, like 30 seconds.

This could be improved by using [circuit breaker](https://donatas.net/blog/2017/03/01/powerdns-pipebackend/) between the server and the client.

A further step to achieve more granular stickiness would be to use GeoDNS service e.g.: PowerDNS. It has full support for [MaxMind legacy GeoIP](https://dev.maxmind.com/geoip/legacy/downloadable/) and [GeoIP2](https://dev.maxmind.com/geoip/geoip2/downloadable/). It could return address by country, city, continent. Just keep in mind that GeoIP _dat_ format is not maintained anymore. Consider using _mmdb_ format. PowerDNS looks up what to return by source IP of the resolver or EDNS - if forwarded.

If your DNS server is dual-stacked, make sure you use both families of GeoIP data (IPv4 and IPv6). Otherwise, it will respond with surprising results.

You could even craft your own mapping conditions on how to respond to arbitrary queries. Like Facebook has offline DNS map cartographer service. The simplest way is to write in CVS and export to dat/mmdb.

But again this does not guaranty high availability if the backend is down.

Here come CDN providers. Put your website under CDN, cache as much as possible masking backend failures. Some CDN providers have their own load balancers thus you should not care much about how they route the traffic to your website. Again, that’s not free from failures.

Instead of using CDN load balancer, implement anycast over more locations to spread the traffic to the nearest location. Sometimes one is better than many. I refer one as a single anycast address which is deployed between a few locations.

I must mention that $anycast is expensive to deploy because maintenance and monitoring are hard. In addition, anycast is not always end-user friendly because of not very fair routing. Sometimes even de-tour occurs in the path from the source to destination.

You should allocate the whole _/24_ block for it (yeah, this is the smallest prefix length over global BGP table). So if you are planning to use only a few addresses from the whole /24 block - that's not the way to go.

If you have only a few locations then no point to use anycast globally. Too much headache.

Even though you have quite enough PoPs, you must ensure your whois information is up-to-date because some ISPs generates route maps according to _inetnum_ country field. Hence a lot of monitoring and traffic engineering stuff are involved.

One more interesting approach is using CDN in front and anycast at the backends. In this case, CDN provider will load balance your traffic using anycast address, but again you are not sure if your content is served from the right region.
To improve this setup we can install GeoDNS server and ask CDN's resolver to query CNAME record instead of A/AAAA. GeoDNS, in turn, will respond with appropriate backend's address according to resolver's (or EDNS) source IP.

>As I mentioned above IPv4 anycast is an expensive solution, but not for an old good friend - IPv6!

In this blog post, I would like to explain sophisticated (not for networking guys) approach on how to handle failovers gracefully. With IPv6 things change. As drawn in the diagram you should see that every PoP has two prefixes announced.

![ipv6-geodns-failover](/images/ipv6-geodns-failover.png)

One global anycast plus region allocated prefix. Both are overlapping prefixes which allow having smooth failover if one region goes down completely. For instance, your GeoDNS server responds to CNAME record with IP _2A02:4780:C3::1_ for the CDN's resolver and at that moment this region is down. New connections will be redirected to the shortest AS-PATH PoP because of global anycast overlapped network.

You should deploy this setup if you care about the infrastructure - it allows you to turn off the whole PoP (or DC) for testing (or Chaos engineering) purposes. Everything you are not testing is breaking.

>Chaos engineering will allow you to personally meet all of your colleagues within a short time – whether you want to or not!
