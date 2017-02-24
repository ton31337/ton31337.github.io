---
layout: post
title: $Cisco iBGP IPv6 sessions for the win
categories:
- blog
---

Today I spent few hours to figure out this hell. 

## TL;DR;

If you want to use IPv6 iBGP sessions with older Cisco IOS versions (in this case it's a Cisco 6500 series device).. Come on, you just can't. Why? Here is the explanation:

```
10:28:18.428588 IP6 (class 0xc0, hlim 64, next-header TCP (6) payload length: 174) 2a02:4780:1:2::2.44410 > 2a02:4780:1:2::1.179: Flags [P.], cksum 0xb9b3 (correct), seq 130:284, ack 321, win 15008, length 154: BGP, length: 154
  Update Message (2), length: 93
    Multi-Protocol Reach NLRI (14), length: 44, Flags [OE]:
      AFI: IPv6 (2), SAFI: Unicast (1)
===== >>      nexthop: 2a02:4780:1:2::2fe80::ce37:abff:febd:ef70, nh-length: 32, no SNPA
        2a02:4780:bad::/48
      0x0000:  0002 0120 2a02 4780 0001 0002 0000 0000
      0x0010:  0000 0002 fe80 0000 0000 0000 ce37 abff
      0x0020:  febd ef70 0030 2a02 4780 0bad
    Origin (1), length: 1, Flags [T]: IGP
      0x0000:  00
    AS Path (2), length: 0, Flags [TE]: empty
    Multi Exit Discriminator (4), length: 4, Flags [O]: 0
      0x0000:  0000 0000
    Local Preference (5), length: 4, Flags [T]: 100
      0x0000:  0000 0064
```

What the hell is "nexthop: 2a02:4780:1:2::2fe80::ce37:abff:febd:ef70"? Expected to have something like this:

```
10:28:18.561391 IP6 (class 0xc0, hlim 255, next-header TCP (6) payload length: 95) 2a02:4780:1:2::1.179 > 2a02:4780:1:2::2.44410: Flags [P.], cksum 0xa0d0 (correct), seq 321:396, ack 284, win 16101, length 75: BGP, length: 75
  Update Message (2), length: 75
    Origin (1), length: 1, Flags [T]: IGP
      0x0000:  00
    AS Path (2), length: 0, Flags [T]: empty
    Multi Exit Discriminator (4), length: 4, Flags [O]: 0
      0x0000:  0000 0000
    Local Preference (5), length: 4, Flags [T]: 100
      0x0000:  0000 0064
    Multi-Protocol Reach NLRI (14), length: 28, Flags [O]:
      AFI: IPv6 (2), SAFI: Unicast (1)
===== >>      nexthop: 2a02:4780:1:2::1, nh-length: 16, no SNPA
        2a02:4780:1::/48
      0x0000:  0002 0110 2a02 4780 0001 0002 0000 0000
      0x0010:  0000 0001 0030 2a02 4780 0001
```

It's quite clear $Cisco bug. In this case we are doing workaround by implementing eBGP sessions with private AS numbers to bypass link-local addresses. If you use eBGP, then `next-hop-self` just skips link-local addresses and doesn't bundle them into peer's address.
