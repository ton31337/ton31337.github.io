---
layout: post
title: How to accidentally blackhole the whole IPv6 block?
categories:
- blog
---

We had an issue where the whole IPv6 block _/32_ was blackholed due to _fat-fingers_ syndrome. The mask for the prefix was accidentally missed and IPv6 traffic just went off.
![missed ExaBGP IPv6 prefix mask](/images/exabgp_ipv6_missed_mask.png)

Almost everything was fine for a long time because there were some more specific routes advertised from other locations with _/48_. But because one of our ISPs (multi-homed) didn't announce one of missing _/48_, thus it was routed (actually blackholed) via the unexpected node.

We cannot fix all the problems people do (_fat-fingers_), but a trivial fix to make sure others do not step on the same mine is nice to have.
![default ExaBGP IPv6 prefix mask](/images/exabgp_ipv6_prefix_mask.png)
