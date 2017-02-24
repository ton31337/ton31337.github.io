---
layout: post
title: Abort on overflow
categories:
- blog
---

After some analyzing kernel's source, I figured out what this actually do. If it can't create the child socket it just goes to `listen_overflow`. If ACK packet is valid, it tries to create `skb` and move socket to `ESTABLISHED` state. If it fails, it returns NULL if `tcp_abort_on_overflow` is disabled (default value), else it sets this packet as ACK and returns NULL. 

I decided to verify how it really works. I've set up a custom web server with listen maximum backlog 5 and ran `ab` tool to generate traffic (to overflow backlog). 

#### tcp_abort_on_overflow=0
```
% ab -n 300 -c 100 http://X.X.X.X/
This is ApacheBench, Version 2.3 <$Revision: 1604373 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking X.X.X.X (be patient)
Completed 100 requests
Completed 200 requests
apr_pollset_poll: The timeout specified has expired (70007)
Total of 294 requests completed
```

#### tcp_abort_on_overflow=1
```
% ab -n 300 -c 100 http://X.X.X.X/
This is ApacheBench, Version 2.3 <$Revision: 1604373 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking X.X.X.X (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Finished 300 requests

Server Software:
Server Hostname:        X.X.X.X
Server Port:            80

Document Path:          /
Document Length:        13 bytes

Concurrency Level:      100
Time taken for tests:   4.630 seconds
Complete requests:      300
Failed requests:        61
   (Connect: 0, Receive: 0, Length: 61, Exceptions: 0)
Total transferred:      23183 bytes
HTML transferred:       3107 bytes
Requests per second:    64.80 [#/sec] (mean)
Time per request:       1543.185 [ms] (mean)
Time per request:       15.432 [ms] (mean, across all concurrent requests)
Transfer rate:          4.89 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  823 1100.2    116    4437
Processing:     4  100 273.5     46    3337
Waiting:        0   42  57.4     15     246
Total:         66  923 1128.7    317    4555
```

As you noticed the first test just failed, due to overflowed backlog queue, the second passed without any burst.

More information about service backlog you can find in my another blog [post](https://sysdig.com/blog/monitoring-memcached-and-socket-queues-with-sysdig/).
