---
layout: post
title: Prevent route leaks by explicitly defining policy
categories:
- blog
---

Route leaks or even hijacks are one of the biggest flaws in global routing.

What are the route leaks? Literally, there are prefixes announced accidentally due to wrongly configured import/export filters. Or those filters do not exist at all.

If a customer or a peer sends routes outside his scope and a provider accepts them, we call those events as route leaks.

Route leaks are more referred to accidentally events, while hijacks are an illegitimate takeover of IP addresses by corrupting routing tables. In such a case, scammers can route the traffic as he wishes. To avoid the aforementioned events providers MUST take care of strict filters what to accept from peers and customers.

Most known filters are:

|                          | Easy to implement | Major vendors support |                                                                               Pros                                                                              |                                                                           Cons                                                                           |
|:------------------------:|:-----------------:|:---------------------:|:---------------------------------------------------------------------------------------------------------------------------------------------------------------:|:--------------------------------------------------------------------------------------------------------------------------------------------------------:|
|           RPKI           |                   |           X           |                           Accept valid prefixes from peers and customers. No need to define any prefix-lists and other ACLs.                          | Requires external validators.  Takes more time for peers to implement.  Not much precise at the moment because peers do not update ROA records properly. |
|        prefix-list       |         X         |           X           |                                     Accepts only defined ranges of prefixes. Allows specifying a dynamic prefix mask (ge, le).                                    |                                                           Can’t filter out prefixes by AS-PATH.                                                          |
|      distribute-list     |         X         |           X           |                                                              Same as prefix-list, but less mature.                                                              |                              Can’t filter out prefixes by AS-PATH.  Cannot assign dynamic prefix mask as prefix-list does.                             |
|   as-path access-lists   |         X         |           X           |                                                                  Filter routes by AS-PATH only.                                                                 |                                                             Cannot filter routes by prefixes.                                                            |
|      maximum-prefix      |         X         |           X           |                                     Set maximum prefix number for a peer or a customer to avoid overfilling the Adj-RIBs-In.                                    |                             This is too much aggressive if one side decides to restart the session under some circumstances.                             |
| WHOIS database filtering |                   |                       |                   Accept prefixes only defined in WHOIS database (RIPE, APNIC, etc.). This is more accurate at the moment comparing with RPKI.                  |                Takes more time to converge. Changes are visible typically in 24 hours. Requires external services to update the control plane.               |
|          RFC8212         |         X         |                       | Accept and/or announce prefixes only if one of the aforementioned filtering techniques applied.  This is bidirectional forwarding due to filtering both directions. |                                                      Only FRRouting at the moment supports this RFC.                                                     |

Another interesting approach I found is by implementing roles where each neighbor defines the role: customer, peer, internal, provider. This approach appends `BGP Open message` to establish an agreement of the relationship of two neighbors. Propagated routes are then marked with iOTC (The Internal Only To Customer)] attribute according to agreed relationship allowing prevention of route leaks.

More about that you can read on [htt ps://tools.ietf.org/html/draft-ietf-idr-bgp-open-policy-02](https://tools.ietf.org/html/draft-ietf-idr-bgp-open-policy-02).

We discussed this draft in short in the FRRouting group, but at the moment there is nothing to implement while it's not released as RFC. And this draft
is questionable if it would give any reasonable effect right now. It would take a huge amount of time to implement for both sides. It's absolutely fine if you use only [FRRouting daemons](https://frrouting.org/), but what about a vendor-agnostic solution?

Instead of using roles in updates and open messages, I found [https://tools.ietf.org/html/rfc8212](https://tools.ietf.org/html/rfc8212) which sounds reasonable to implement.

The RFC defines that both peering sides should require import/exporter filters explicitly defined. What does it mean? By default, all(?) vendors do not require route-map to be configured for a neighbor.

The snippet below will announce everything it has in its _Adj-RIB-Out_ for neighbor 192.168.3.1.

```
router bgp 65031
 neighbor 192.168.3.1 remote-as 65032
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
!
```

[Here](https://github.com/FRRouting/frr/pull/3746) we have a new BGP `bgp ebgp-requires-policy` knob which requires filters (filter-list, distribute-list, prefix-list or route-map defined) for every eBGP session.

![](/images/rfc8212.gif)

```
router bgp 65031
 bgp ebgp-requires-policy
 neighbor 192.168.3.1 remote-as 65032
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor 192.168.3.1 prefix-list exit4-in in
  neighbor 192.168.3.1 route-map exit4-out out
 exit-address-family
!
ip prefix-list exit4-in permit 10.0.0.0/24
!
route-map exit4-out permit 10
!
```

To clarify, if we remove inbound prefix-list from this neighbor, we will receive zero prefixes due to default behavior.

```
router bgp 65031
 bgp ebgp-requires-policy
 neighbor 192.168.3.1 remote-as 65032
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor 192.168.3.1 route-map exit4-out out
 exit-address-family
!
route-map exit4-out permit 10
!
```

Adj-RIB-In table for neighbor 192.168.3.1 will be equal to zero prefixes. It prevents accepting any prefixes if the policy is forgotten to define explicitly.

Even though we see an informational warning that _Inbound updates discarded due to missing policy_ under `show bgp neighbors 192.168.3.1`.

Hence, instead of just reading the documentation, it’s more fun to play archeologist and try to implement this into real-life examples. An attempt to cool again.
