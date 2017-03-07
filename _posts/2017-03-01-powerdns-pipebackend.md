---
layout: post
title: 'Razor: Scalable service for randomizing DNS requests'
categories:
- blog
---

At first, at Hostinger we looked for some production ready solution, but nothing fit our needs. There was an idea to write our own service, but eventually, it was rejected due to complexity. Later, we started to analyze current DNS services to see if they can be used as modular services with custom backends. Fortunately, [PowerDNS](https://www.powerdns.com/) has an option to implement custom backend.

Next step was to pick service as the backing store for DNS records. This move was fast and the winner was [Redis](http://redis.io/). Why Redis? Because it already has [SRANDMEMBER](http://redis.io/commands/srandmember) command. MySQL, for instance, should call `SELECT content FROM records ... ORDER BY rand()` or something like that, which is absolutely slow operation comparing with `SRANDMEMBER` O(1).

Once we have two main components for building `Razor`, we have to select the language to connect those components smoothly. We tested with few languages including Ruby, Python, Crystal, but numbers are very different. I want to share this [post](http://www.stefanwille.com/2015/05/redis-clients-crystal-vs-ruby-vs-c-vs-go/) which is self-defining. Hence we decided to continue with [Crystal](http://crystal-lang.org/).

#### More details

We call our entries in Redis routes. Each route is associated with pod: `<zone>.route-<pod_id>.000webhost.awex.io`, for example: `us-east-1.route-1.000webhost.awex.io`. This route is CNAMEd with client's application name.

If we want to add/remove IP addresses from being returned as end-point for application, we just modify Redis keys. Redis key looks like: `<route>:<type>`, for example: `us-east-1.route-1.000webhost.awex.io:A` for IPv4 or `us-east-1.route-1.000webhost.awex.io:AAAA` for IPv6.

As a result, we are able to handle ~80.000 requests per second with one VM. For scaling this service to end users we run it in different regions including United States and United Kingdom at the moment. We announce the same anycast prefix from all these regions to satisfy user's request as fast as possible. Inside region, we use anycast inside and ECMP together to load balance DNS traffic and avoid saturation for one node.

After we deployed this service in production, we got many complains about `DNS_PROBE_FINISHED_NXDOMAIN` returned by browser. Started to dig into this problem and saw, that we randomly return `status: REFUSED`.

```
probe process("/usr/sbin/pdns_server").function("getSOA@/usr/src/debug/pdns-4.0.3/pdns/dnsbackend.cc:217").return
{
    domain_id = $sd->domain_id;
    if (domain_id < 0 && $p->qtype->code == 6)
        printf("%d\n", $return);
}

output:
0
0
0
0
1
```

As you see a lot of queries respond with false to `getSOA()`. That was not good, because we use [pipe-regex](https://doc.powerdns.com/md/authoritative/backend-pipe/#pipe-regex) to filter out not necesarry queries for PIPEBackend. Hence, we expect to see only valid queries. PowerDNS returns `REFUSED` if `SOA` record is not returned. First I enabled logging for queries and responses. This is what I got:

```
Q        route-1.000wEBhost.AWex.io        IN        SOA        -1        85.62.233.82
```

rAnDOm gENERatEDE qnames, and looks like this is [not handled](https://github.com/PowerDNS/pdns/blob/master/modules/pipebackend/pipebackend.cc#L179) by PowerDNS before sending query to PIPEBackend. It would be [great to have](https://github.com/PowerDNS/pdns/issues/5088) a configuration option for PowerDNS like `pipe-lowercase`, but we fixed it quickly by converting to lowercase in [our backend](https://github.com/ton31337/pdns_razor/blob/master/razor.cr#L23).

#### Takeaways

* Do not reinvent the wheel, use current open source products with extension in mind;
* Crystal is amazing and super fast language.
