---
layout: post
title: Unnecessary Updates in BGP
categories:
- blog
---

It is well known that BGP, as a distance-vector protocol, suffers from path exploration: For every withdrawn route (AS path or any other mandatory attribute change), the next best, supposedly valid route is selected and announced, until there are no more candidates left in the router's RIB.

BGP routers can generate multiple identical announcements with empty community attributes if stripped at egress. This is an undesired behavior. Why do we need to send an UPDATE if this is not an actual change?

Imagine a topology below:
![BGP Unnecessary Updates](/images/bgp-unnecessary-updates.png)

Assume a converged network. In order to induce updates for prefix _p_, we disable the link between Y1 and Y2 and wait for arriving updates at collector C1.

I have FRRouting configuration at X1:

```
router bgp 65002
  no bgp ebgp-requires-policy
  neighbor 10.0.1.1 remote-as external
  neighbor 10.0.1.1 timers 3 10
  neighbor 10.0.2.2 remote-as external
  neighbor 10.0.2.2 timers 3 10
  address-family ipv4 unicast
    redistribute connected
    neighbor 10.0.1.1 route-map c1 out
  exit-address-family
!
bgp community-list standard c1 seq 1 permit 65004:2
bgp community-list standard c1 seq 2 permit 65004:3
!
route-map c1 permit 10
  set comm-list c1 delete
!
```

X1 receives paths with communities 65004:2 and 65004:3 from Y1 because those paths are tagged at Y2 and Y3 appropriately.

When X1 sends the best path to C1 it generates duplicate updates because of attribute change (community, 65004:2 -> 65004:3). But since we stip all communities at egress, it's absolutely not necessary sending duplicates towards C1 because the Adj-Rib-Out wasn't changed.

I developed a [patch](https://github.com/FRRouting/frr/pull/7507) for FRRouting, which prevents sending duplicate updates if the path actually not changed. This is valid for all attributes.

For more detailed information and experiments, please read at [https://www.cmand.org/communityexploration/](https://www.cmand.org/communityexploration/). You will find more configuration examples for other routers (Cisco, Juniper, Nokia) and other software routing daemons (Bird, OpenBGPD, Quagga).
