---
layout: post
title: Measure TCP metrics LD_PRELOAD-ish way
categories:
- blog
---

Why `LD_PRELOAD`?

Because most of the services do not expose `TCP_INFO` about the socket natively. So there are two ways, one is harder, one is trivial. I will talk about the latter one.

LD_PRELOAD allows you to intercept syscalls, grab necessary data and return back to normal processing. In turn, LD_PRELOAD can be easily injected using environment variable `LD_PRELOAD=` (globally or per-service), linked statically or whatever you want.

Here is the basic example to interept `accept()` syscall, get TCP metrics and return to the real syscall:

```
/*
# gcc -shared -fPIC accept.c -o accept.so -ldl
# LD_PRELOAD=/usr/share/accept.so /usr/local/bin/service
*/

#define _GNU_SOURCE
#include <sys/syscall.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <stdio.h>
#include <stdint.h>
#include <dlfcn.h>
#include <netinet/tcp.h>

int (*orig_accept)(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)
{
        struct tcp_info tcp;
        int len = sizeof(info);
        char *logfile = "/var/log/pdns/rtt.log";
        FILE *fp;
        int fd;

        setbuf(stdout, NULL);

        orig_accept = dlsym(RTLD_NEXT, "accept");
        fd = orig_accept(sockfd, addr, addrlen);

        if (getsockopt(fd, IPPROTO_TCP, TCP_INFO, (void *)&tcp, (socklen_t *)&len) == -1)
                printf("error: getsockopt()\n");

        fp = fopen(logfile, "a+");
        if (fp == NULL)
                printf("error: cannot open %a\n", logfile);

        fprintf(fp, "%u\n", tcp.tcpi_rtt / 1000);
        fclose(fp);

        return fd;
}
```

To verify if it's working or not just type:

```
# grep accept.so /proc/26663/maps
7fb375525000-7fb375526000 r-xp 00000000 08:01 50341026                   /usr/share/accept.so
7fb375526000-7fb375725000 ---p 00001000 08:01 50341026                   /usr/share/accept.so
7fb375725000-7fb375726000 r--p 00000000 08:01 50341026                   /usr/share/accept.so
7fb375726000-7fb375727000 rw-p 00001000 08:01 50341026                   /usr/share/accept.so
```

#### Playground

It's always useful to measure performance before/after changes to see if it was worth your time. I used this technique to evaluate the effect before the switch to anycast-based DNS solution. 

But wait, DNS works in UDP mostly ;-) Yes, it works unless data size exceeds 512 bytes of response. Hence I employed this hackish method to measure DNS using TCP protocol because with UDP it's not possible due to its nature:

* UDP doesn't have 3WHS, which means it's not possible to measure RTT;
* It would be possible to measure this if clients would use `SO_TIMESTAMP` for sending data and compare at receiving side.

##### Results

```
# cat /var/log/pdns/rtt.log | python mmhistogram
Values min:0.00 avg:56.07 med=16.00 max:64000.00 dev:514.34 count:2912957
Values:
 value |-------------------------------------------------- count
     0 |                                                   1087
     1 |                                                   97
     2 |                                                   297
     4 |                                                   955
     8 |************************************************** 1503390
    16 |                    ****************************** 908123
    32 |                                               *** 110458
    64 |                                         ********* 287174
   128 |                                                 * 52383
   256 |                                                   28575
   512 |                                                   6807
  1024 |                                                   5729
  2048 |                                                   3534
  4096 |                                                   2191
  8192 |                                                   1430
 16384 |                                                   713
 32768 |                                                   14
```
