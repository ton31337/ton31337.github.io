---
layout: post
title: Messaging bus using multicast
categories:
- blog
---

Multicast, in short, is one-to-many or many-to-many IP communication mechanism. You can compare it with publish-subscribe principle. In my experience, I never met people who understand and uses multicast in production, neither trying it to understand, because it looks like a black hole. But don't be afraid of it, it's nothing more special like anycast or broadcast or even unicast. Of course, it's harder to implement it and use it (especially debugging) properly in heavy production environments, which requires multicast routing (nobody understands how it works :). 

Multicast is mostly used by network stuff (e.g. router discovery in dynamic routing protocols) or service discovery by services (e.g. ElasticSearch). I never saw multicast had ever used for other purposes like messaging bus? Let's see what I meant.

Have you ever used the publish-subscribe mechanism in ZeroMQ, RabbitMQ, Redis, etc? They have this implementation on top of Layer7. So why not go back in the 90s and do not reuse well-known Layer3 for this routing task? I don't say it fits all cases. I want to cover only message bus which works as fanout exchange.

Let's say I have one publisher which listens on an arbitrary multicast group and sends some data to this group.

Sender (aka. master):

```
require 'socket'

MULTICAST_ADDR = "225.4.5.6"
PORT= 5000

begin
  socket = UDPSocket.open
  socket.setsockopt(Socket::IPPROTO_IP, Socket::IP_TTL, [1].pack('i'))
  loop do
    socket.send(send_some_stuff(), 0, MULTICAST_ADDR, PORT)
    sleep 0.1
  end
ensure
  socket.close
end
```

Receiver (aka. slave):

```
require 'socket'
require 'ipaddr'

MULTICAST_ADDR = "225.4.5.6"
PORT = 5000
TTL = 1

member = IPAddr.new(MULTICAST_ADDR).hton + IPAddr.new('0.0.0.0').hton

server = UDPSocket.new
server.setsockopt Socket::IPPROTO_IP, Socket::IP_ADD_MEMBERSHIP, member
server.setsockopt Socket::IPPROTO_IP, Socket::IP_MULTICAST_TTL, TTL
server.setsockopt Socket::IPPROTO_IP, Socket::IP_TTL, TTL
server.bind Socket::INADDR_ANY, PORT

loop do
  data, info = server.recvfrom(1024)
  do_something_with_received_data(data)
end
```

This is the simplest case when you don't need any special routing of messages, just distribute them across all listeners, but the principle itself is quite good and simple.

#### Conclusion
* As always, use simple tools for simple tasks;
* If you think good architecture is complex, try bad architecture.
