---
layout: post
title: Scaling Fuck Transfer Protocol
categories:
- blog
---

What is FTP? Is it still alive?!

Unfortunately, yes. In general FTP protocol isn’t so cool and scalable as HTTP/HTTPS and I think it will die soon. Linux Kernel already [pissed](https://www.kernel.org/shutting-down-ftp-services.html) it off. I strongly recommend people to move this way as well.

Let's see few cases how FTP is _scaled_.

The most trivial use case is running standalone servers. This is what every idiotic _$end_user_ without any fundamental understanding imagines this. 
![FTP-standalone](/images/standalone_ftp.png)

Another more robust and resilient to failures mode is running FTP proxy in front of standalone servers. This is mode allows shedding some load from backends like authentication, caching, etc., thus increasing consistency and usability for users. For instance, keep using a single domain for all the users instead of pushing any random domain for every server.
![FTP-Prody](/images/ftp_proxy.png)

Ok, sounds nice, but not so nice for shits cleaners aka. SREs (Site Reliability Engineers) - count me in. What if we just add more than one FTP proxy in front to absorb clients traffic? With HTTP(S) or other stateless protocol, it's trivial, you can load balance requests using DNS round-robin or the more sophisticated method using anycast. Almost all of the people who never deployed anycast say it's so cool and easy to deploy. Relax dudes, it's easy only on the slides, complexity comes maintaining it and debugging. With IPv6 this is even harder due to ICMv6 and MTU.
![FTP-Multi-Proxy](/images/ftp_multi_proxy.png)

The first idea was just to use [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) with [client’s IP](https://tools.ietf.org/html/draft-vandergaast-edns-client-ip-01). But there will be problems with clients under carrier-grade NATs where outgoing IP addresses are changing over time. 

The second idea came to use consistent hashing with client’s subnet. For IPv4 use first 24-bits and for IPv6 first 48-bits. This allows keeping consistent DNS answers to the same clients. The snippet below is taken from [razor](https://github.com/ton31337/pdns_razor) pipe-backend for [PowerDNS](https://www.powerdns.com/).

```
  private def ipv4_int(ip)
    ip_int = 0.to_big_i
    ip.split(".").each_with_index do |oct, i|
      ip_int |= oct.to_big_i << (32 - 8 * (i + 1))
    end
    ip_int
  end

  private def ipv6_decompress(ip)
    ip_arr = [] of String
    splitted = ip.split("")
    compressed = 8 - ip.split(":").size
    splitted.size.times do |i|
      ip_arr << splitted[i]
      if splitted[i] == ":" && splitted[i+1]? == ":"
        compressed.times do |_|
          ip_arr << "0" << ":"
        end
        ip_arr << "0"
      end
    end
    ip_arr.join
  end

  private def ipv6_int(ip)
    ip_int = 0.to_big_i
    ipv6_decompress(ip).split(":").each_with_index do |word, i|
      ip_int |= word.to_big_i(16) << (128 - 16 * (i + 1))
    end
    ip_int
  end

  private def ip_hashed(ip, count)
    if ip.includes?(":")
      (ipv6_int(ip) >> (128 - 48)) % count
    else
      (ipv4_int(ip) & 0xffffff00) % count
    end
  end
```

I launched this behavior on my test bed and it works as expected. And got another problem: how should DNS server answer to the client while using different IP pools for different servers (server has many IPv4/IPv6 addresses)? How to distinguish them? Rescue to the problem - groups (pools). I created separate groups with separate IP address pools and I do group consistent hashing to return random IP address from the appropriate group, which is selected by aforementioned client's subnet bits. 

#### Conclusion

Kill it. Completely.
