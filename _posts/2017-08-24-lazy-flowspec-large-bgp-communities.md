---
layout: post
title: Lazy flowspec using large BGP communities
categories:
- blog
---

Almost every ISP responds unfortunately, they still cannot handle flowspec standard. It's nearly 2018, almost every BGP-aware daemon software is able to send/receive flowspec. Those who don't know what flowspec is:

>The BGP flow specification (flowspec) feature allows you to rapidly deploy and propagate filtering and policing functionality among a large number of BGP peer routers to mitigate the effects of a distributed denial-of-service (DDoS) attack over your network.

In other words, flowspec allows you to filter out packets more granular than with RTBH (remotely triggered blackhole). Flowspec is sent using another one NLRI (network layer reachability information) in BGP protocol.

So, what to do for guys if they do not support flowspec? You may ask them if they support large BGP communities. Most of vendors [still](http://largebgpcommunities.net/implementations/) have this implementation in their roadmaps, but to show an example how it works is fair enough using ExaBGP (my one of the favorite daemon). 

With regular BGP communities, this wouldn't be possible because it's bound to 4-bytes while with large communities it can be extended to 12-bytes long.

For this lazy flowspec I will use lazy example because it's just an interesting way to reuse BGP large communities. In this example, I use ExaBGP as mentioned above to announce and to receive BGP communities.

Here is the snippet (written with my favorite programming language as usually) which could be used to announce `large-community`:

```
require 'ipaddr'

# Define firewall rules [source, destination, port]
flowspecs = {
  block_443: ['10.0.0.1', '192.168.0.1', 443],
  block_28501: ['10.0.0.1', '192.168.0.2', 28501]
}

def communities(flowspecs)
  flowspecs.map do |_, flow|
    format("%d:%d:%d", IPAddr.new(flow[0]).to_i, IPAddr.new(flow[1]).to_i, flow[2])
  end
end

while true
  $stdout.write format("announce route 10.66.66.66/32 next-hop 10.66.66.254 large-community [#{communities(flowspecs).join(' ')}]")
  $stdout.flush
  sleep 5
end
```

The idea is to use a single route (could be multi route solution, but it's not needed) and append communities (firewall rules) as much as needed for this route.

You could ask, how about IPv6? Nohow. 4-bytes is not enough to store IPv6 address. That's why I call it lazy ;-)

Ok, an example shown above is for sending notes to another end (customer edge). The next example will be for chewing those notes accordingly. Actually, almost the same code path could be reused from [exa-template](https://github.com/ton31337/exa-template/blob/master/lib/exa-template.rb#L48-L59) with some minor modifications:

```
def parse_services(event)
  community = event['neighbor']['message']['update']['attribute']['large-community']
  community&.each do |c|
    src, dst, port = c.split(':').map do |level|
      level.to_i > 65535 ? IPAddr.new(level.to_i, Socket::AF_INET).to_s : level
    end
    generate_iptables_rule(src, dst, port)
  end
end
```

The snippet above just parses `large-community` and passes variables to tail-call to generate iptables rules. This appropriately will block all traffic to ports 443/28501 from 10.0.0.1 to 192.168.0.1/192.168.0.2.

#### Conclusion

* BGP large communities offer a more granular way to play with routes (traffic engineering);
* Some problems simply are not worth to be solved, but flowspec MUST be swapped with RTBH in nearly future!
