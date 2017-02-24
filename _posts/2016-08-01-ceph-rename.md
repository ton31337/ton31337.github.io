---
layout: post
title: CephFS, quotas and rename()
categories:
- blog
---

Ceph like always at its best without proper documentation. The best documentation for Ceph is just reading the source code. This case is not an exception. Take this simple example:

```
#include <stdio.h>
int main()
{
        int rc = -1;
        char *renamefrom = "/home/003/10003/test1.jpg";
        char *name = "/home/003/10003/public_html/test.jpg";
        if (rc = rename(renamefrom, name) < 0) {
                printf("%d\n", rc);
        }
        return 0;
}
```

It tries to rename file between different directories. It doesn't work, such wow.

ceph-fuse client code says:
```
int Client::_rename(Inode *fromdir, const char *fromname, Inode *todir, const char *toname, int uid, int gid)
...
if (cct->_conf->client_quota &&
      fromdir != todir &&
      (fromdir->quota.is_enable() ||
       todir->quota.is_enable() ||
       get_quota_root(fromdir) != get_quota_root(todir))) {
    return -EXDEV;
}
...
```

Ceph-fuse handles EXDEV strictly, thus it doesn't allow to rename file between different directories if quota is enabled for one of them or they should be equal.
