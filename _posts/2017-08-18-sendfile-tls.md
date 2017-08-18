---
layout: post
title: sendfile() and TLS
categories:
- blog
---

Every skilled sysadmin knows about [sendfile()](http://man7.org/linux/man-pages/man2/sendfile.2.html) syscall. 

I remember my first proof of concept taken in 2012 where I tried to compare real `cp` (old coreutils package ~2000) command and a custom simple wrapper (do not pay too much attention on the code - just take an idea):

```
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sched.h>

enum flags_t { NO_ERR, ERR_OPEN, ERR_STAT };

int cp(const char *src, const char *dst, int *err)
{
  struct stat src_stats;
  ssize_t copied = 0;

  int rfd = open(src, O_RDONLY);
  if(!open)
    *err |= ERR_OPEN;

  int r = fstat(rfd, &src_stats);
  if(r == -1)
    *err |= ERR_STAT;

  if(*err)
    return 1;

  int wfd = open(dst, O_RDWR|O_CREAT|O_TRUNC,
         src_stats.st_mode & S_IRWXU | src_stats.st_mode & S_IRWXG | src_stats.st_mode & S_IRWXO);
  copied = sendfile64(wfd, rfd, NULL, src_stats.st_size);

  close(rfd);
  close(wfd);

  if(copied)
    return 0;

  return 1;
}

int main(int argc, char *argv[])
{
  int err = 0;
  if(argc == 3) {
    if(cp(argv[1], argv[2], &err))
      printf("Error number: %d\n", err);
    return 0;
  } else {
    printf("Missing files\n");
    return -1;
  }
  return 0;
}
```

I do not remember test results, but it was way better with `sendfile()` instead of using `read()/write()` combination. sendfile() does copy within the kernel so bypassing users-pace transfers. There is one limitation with sendfile() that it can copy up to 2GB of data. You must use [sendfile64()](https://linux.die.net/man/2/sendfile64) syscall for larger files if needed.

But let's talk about more interesting stuff with `sendfile()` and TLS. Netflix [introduced](https://people.freebsd.org/~rrs/asiabsd_2015_tls.pdf) new technique to process TLS within the kernel, because sendfile() doesn't allow to enter user-space, which means data encryption/decryption must be done within the kernel.  

The main idea is that they designed this approach by leaving TLS sessions inside the application (user-space TLS library does the heavy lifting) while encryption is inserted into the sendfile data pipeline.

sendfile() will read the file from disk, encrypt chunks and blow through the out_fd, which is in most cases network socket. Without TLS in the kernel, only userspace applications are able to do this, hence sendfile() is not compatible with TLS. 

Facebook touched similar approach regarding crypto kernel TLS socket support (kTLS) as well. Facebook is doing this by creating another one [kTLS socket](https://github.com/ktls/af_ktls), for instance:
```
if (read(...) < 0) {
  getsockopt(ktls_fd, AF_KTLS, KTLS_GET_IV_RECV, ...);
  SSL_read(tcp_fd, ...);
}
```
Application communicastes with kTLS socket where kTLS communicates directly with sendfile() and friends. 

#### Conclusion

If these will come to the front it will be possible to do even layer7 proxying directly within the kernel \o/
