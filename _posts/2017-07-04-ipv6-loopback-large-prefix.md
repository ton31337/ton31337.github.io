---
layout: post
title: ::1/128 enlarge to not to enlarge?
categories:
- blog
---

IPv4 has by design `127.0.0.0/8` delegated for loopback usage. It means that you can use ~16M addresses to identify hosts inside your fleet. I've touched this usage in networking world, but who else really needs this behavior while we are living in 2017 (containers world)? To be honestly, I'm talking about ancient IPv4 protocol.

IPv6 doesn't have allocated /64 or so for loopback - it's as simple as it should be `::1/128`. But there is a proposal for [larger loopback prefix](https://tools.ietf.org/html/draft-smith-v6ops-larger-ipv6-loopback-prefix-04) `1::/32`:

>Allocating a /32 prefix for the loopback function may seem excessive, as a /48 length prefix would satisfy the larger loopback prefix requirements. However, within the parent 0000::/8 special purpose prefix, there are approximately 16 million /32 prefixes, so a single /32 for the larger loopback prefix is easily afforded. A /32 larger loopback prefix will satisfy all current and likely future uses of the loopback function.

As a result `1::1/64` is proposed as the best candidate because it's quite easy for humans to remember. And /64 is for EUI-64, just don't break the internet. Also, this should lie down under loopback handling properly as I described in my previous [post](http://donatas.net/blog/2017/04/25/ipv6-loopback/).

If you didn't know, you can ping any address from `127.0.0.0/8` range:

```
$ ping 127.22.23.53 -c1
PING 127.22.23.53 (127.22.23.53) 56(84) bytes of data.
64 bytes from 127.22.23.53: icmp_seq=1 ttl=64 time=0.053 ms
```

The kernel just loops those packets back to the same node.

It depends on your creativity level, but to get the idea what I'm talking about see this example below:

```
require 'socket'

jobs = {
    '127.0.0.22' => 'SSH',
    '127.0.0.53' => 'DNS',
    '127.0.0.80' => 'HTTP'
}

socket = TCPServer.open(31337)

loop do
        connection = socket.accept
        job, _ = connection.local_address.getnameinfo
        connection.puts jobs[job]
        connection.close
end
```
```
$ exit | nc 127.0.0.53 31337
DNS
$ exit | nc 127.0.0.80 31337
HTTP
```
This wasteful range could be exploited in something similar as with [IPv6 ILA](https://www.ietf.org/id/draft-herbert-nvo3-ila-04.txt) but in application level. While ILA works without breaking TCP - almost natively, but with some kind of external control plane for mapping identifier with the locator. 

In 2017 for IPv6 (I suppose you are smart enough and don't ask me about IPv4 anymore) you can take a look for [AnyIP](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ab79ad14a2d51e95f0ac3cef7cd116a57089ba82). It just allows you to bind the whole prefix and respond NS packets. The idea is more-less identical to 127.0.0.0/8.

#### Conclusion

Large IPv6 loopback prefix in some cases would be nice to have, but I can't imagine something really fancy useful. While we are donated with AnyIP, we can tackle loopback problem locally with some arbitrary prefix.
