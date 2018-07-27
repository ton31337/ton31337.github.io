---
layout: post
title: Apply route-map for default-originate in FRRouting
categories:
- blog
---

If you use Clos topology you should be aware that there are no direct links between leaves or spines. So, if you announce the default route unconditionally, you know what happens - traffic destined to `0.0.0.0/0` (or `::/0`) is blackholed. And what happens if you want to announce the default route conditionally? Let's say I need to announce `0.0.0.0/0`
only if an arbitrary community `123:1` exists. For instance:

```
ip community-list standard upstream permit 123:1
!
route-map default-route permit 1
  match community upstream
  set as-path prepend 65031 65031
```

This looks legit and should work as expected, but unfortunately not. FRRouting just creates dummy attributes and uses them instead of applying attributes from the specified route-map. We can double check this using `tcpdump`:

```
12:05:46.149474 IP (tos 0xc0, ttl 1, id 52011, offset 0, flags [DF], proto TCP (6), length 97)
    10.0.0.1.53746 > 10.0.0.2.179: Flags [P.], cksum 0xc2af (correct), seq 108:153, ack 130, win 229, options [nop,nop,TS val 832288 ecr 732246], length 45: BGP
    Update Message (2), length: 45
      Origin (1), length: 1, Flags [T]: IGP
        0x0000:  00
      AS Path (2), length: 6, Flags [TE]: 65031
        0x0000:  0201 0000 fe07
      Next Hop (3), length: 4, Flags [T]: 10.0.0.1
        0x0000:  0a00 0001
      Updated routes:
        0.0.0.0/0

```

Notice `AS Path` attribute. It's not `65031 65031`, but just `65031`. 

As always I tried to debug and verify if it's really hits the actual code in the source:

```
stap -e 'probe process("/usr/lib/frr/bgpd").statement("subgroup_default_originate@/root/frr/bgpd/bgp_updgrp_adv.c:720")
{ rintf("route-map type for default-originate: %d", $bgp->peer_self->rmap_type); }'

route-map type for default-originate: 16
```

16 means `PEER_RMAP_TYPE_DEFAULT`. Looks good.

I wanted to achieve that `set` in route-map would work with `default-originate`.

Configuration is simple:

#### spine
```
router bgp 65031
 bgp router-id 10.0.0.1
 timers bgp 10 30
 neighbor 10.0.0.2 remote-as 65032
 !
 address-family ipv4 unicast
  redistribute kernel
  neighbor 10.0.0.2 default-originate route-map default
 exit-address-family
!
route-map default permit 10
 set as-path prepend 65031 65031
 set metric 200
!
```

#### leaf
```
router bgp 65032
 bgp router-id 10.0.0.2
 neighbor 10.0.0.1 remote-as 65031
!
```

I don't usually have enough time to contribute to the community, but this project is my favorite one, thus why don't invest my time after work and don't try!

### Conclusion

Contribution to community software is awesome. You learn new stuff, you get new ideas from other smart people. It's absolutely worth spending your so expensive time on contributions. What I liked most was that FRRouting uses CI/CD quite heavy including [clang static analysis](https://clang-analyzer.llvm.org/) (which is really cool, I never used it and now I understand how and when to use it).

I personally did tests on [kitchen / vagrant](https://github.com/test-kitchen/kitchen-vagrant).
* [.kitchen.yml](https://github.com/ton31337/frr-default-originate/blob/master/.kitchen.yml)
* [provision.rb](https://github.com/ton31337/frr-default-originate/blob/master/provision.rb)

So finally the [feature request](https://github.com/FRRouting/frr/pull/2708/commits/74401e62721b8f83ff0e34127d6235fda112c7c8
),  landed on the [master](https://github.com/FRRouting/frr/pull/2755/commits/c2e10422033771da9f12a4a283b0bc767240a3d8) in FRRouting:

```
leaf1-debian-9# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0
Paths: (1 available, best #1, table Default-IP-Routing-Table)
  Advertised to non peer-group peers:
  10.0.0.1
  65031 65031 65031
    10.0.0.1 from 10.0.0.1 (10.0.0.1)
      Origin IGP, metric 200, localpref 100, valid, external, best
      AddPath ID: RX 0, TX 16
      Last update: Tue Jul 24 13:20:08 2018
```
