---
layout: post
title: 5-level paging is coming to the reality
categories:
- blog
---

Few [days](http://blog.donatas.net/blog/2017/03/30/whd-2017/) ago I've heard about [Intel Optane](http://www.intel.de/content/www/de/de/architecture-and-technology/intel-optane-technology.html) technology. This is gonna be a good step for hosting providers to provide more resources for the competitive price.

What is tough, that this solution does not require any breaking changes from user-space applications, it just uses the same paging as with regular memory. With current 4-level paging, you have a limit for physical memory up to 64TB. If the market will keep the pace for these devices as with HDDs then you should guess what would happen. Even [DDR5](https://arstechnica.com/gadgets/2017/03/next-generation-ddr5-ram-will-double-the-speed-of-ddr4-in-2018/) is hitting the market. Eventually, this will hit this limit. But.. Linux community is already working on 5-level paging, which will be able to do paging up to 4PB of physical memory.

Obviously, Intel Optane memory itself is slower than regular memory, but for some workloads, it's perfect, even for those who are memory capacity-bound. Facebook told they are hitting a little bit different case, actually memory bandwidth-bound.

When it's time for the CPU scheduler to switch to another process, a context switch is performed. It's a quite expensive operation, because of its current implementation. With every context switch, TLB entries are flushed for the CPU, which means all virtual-physical memory mappings are invalidated. At the moment there is no way to make TLB be smarter and give a hint to avoid flushing the whole TLB entries. Fortunately, there are some rumors to introduce context-based invalidation, e.g.: by adding some sort of context ID in page struct, or flushing only if switched process isn't the same executable.

Hence, basically, when context switch happens, cr3 register for CPU is changed which points to page global directory for a given process. Imagine what would happen if you would have no paging at all. Each process would have too much over wasted space for storing virtual-physical mappings. It would be totally unusable for these days when we have a lot of processes running on the machine.

I have to mention page faults, which translates to "no virtual-physical mapping is found". At the moment there is no callback based syscall which could be able to handle page faults from the application perspective, but `userfaultfd()` is trying to emerge. I think it should get the address which was marked as faulty from cr2 register and reuse somewhere or just continue as usually. This would be great for some cases like live snapshotting for container or so. When the page fault occurs, we would be able to sync that address.

Since Linux kernel uses 4K as page size, it's getting to be more painful for performance. Page faults could be reduced by using huge tables so bypassing some of the middle page tables and avoiding unnecessary translate operations, for instance using 2MB as a page size. Or upcoming Linux kernel releases are discussing even to increase `PAGE_SIZE` to be higher than 4K. But there are other kludges. User-space applications or even `glibc` wrappers are hard coded to 4K as a page size, hence you understand what would happen. By the way with 5-level paging huge pages could be used even as big as 512GB of size.

#### Conclusion

After some introduction how memory management in Linux works, now we are seeing 5-level paging on the mainstream kernel. This is getting to look like real 64-bit architecture (time for a joke). Still, seven most significant bits are reserved, while next 9 bits formed a `P4D` level. This new level is put between page global directory and page upper directory. Hence, this extends physical memory usage up to 4PB, quite enough for a while (actually like with IPv6).
