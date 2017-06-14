---
layout: post
title: Overwhelmed security for scaredy-cats
categories:
- blog
---

Yesterday I read [ip(7)](http://man7.org/linux/man-pages/man7/ip.7.html). While reading I found `IP_TTL` and remembered such a nice feature almost every vendor has for BGP security called `BGP ttl-security check`.

>The BGP Support for TTL Security Check feature introduces a lightweight security mechanism to protect external Border Gateway Protocol (eBGP) peering sessions from CPU utilization-based attacks using forged IP packets. Enabling this feature prevents attempts to hijack the eBGP peering session by a host on a network segment that is not part of either BGP network or by a host on a network segment that is not between the eBGP peers.

>You enable this feature by configuring a minimum Time To Live (TTL) value for incoming IP packets received from a specific eBGP peer. When this feature is enabled, BGP will establish and maintain the session only if the TTL value in the IP packet header is equal to or greater than the TTL value configured for the peering session. If the value is less than the configured value, the packet is silently discarded and no Internet Control Message Protocol (ICMP) message is generated. This feature is both effective and easy to deploy.

Hence, implementation is trivial, just define minimum TTL and expect to see incoming packets with higher or equal TTL value. When it comes to application level security, it's possible to reuse this technique by enforcing application to reject packets if they do not match TTL values.

For instance, let's listen on custom TCP port with `IP_TTL` socket option applied with the value of 1. This means - reject packets with TTL value higher than 1. This is almost the same as an internal network - one hop away. If you operate on leaf-and-spine topology then it works perfectly.

One more real use case would be like I wrote [before](http://blog.donatas.net/blog/2017/04/20/http-request-validation/) when you have a proxy layer and backend layer and want to deny all non-proxied requests then set `IP_TTL` on backend sockets.

#### Proof of concept

Simple ruby TCP server:

```
require 'socket'

socket = TCPServer.open(31337)
socket.setsockopt(Socket::IPPROTO_IP, Socket::IP_TTL, 1)
socket.setsockopt(Socket::SOL_SOCKET, Socket::SO_REUSEADDR, true)

loop do
        Thread.start(socket.accept) do |client|
                client.puts "Connection accepted!"
                client.close
        end
end
```

Connecting from the local network:
```
% nc X.Y.Z.W 31337
Connection accepted!
```

#### Conclusion

* It's always better to have fewer engineers with a full count of fingers;
* I submitted the [patch](https://lists.apache.org/thread.html/27fc46427e5f7aa6b86c5490eb127f0a6d74fecce32d70c62c038749@%3Cdev.httpd.apache.org%3E) to [Apache](http://httpd.apache.org/) which would be one of the best candidates to filter out external requests originating not from the local network.
