---
layout: post
title: MTU issue on vmxnet3 driver with IPv6-only network
categories:
- blog
---

Today I noticed interesting behavior with `vmxnet3` driver on ESXi. Problem is, that driver begins dropping packets after receiving them with MTU higher than 1522 including Ethernet header, checksums, etc., which is the default for NIC. This is happening only with IPv6 packets.

## Some details

#### Guest OS

```
# ip link
2: eno16780032: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000

# ethtool -S eno16780032 | grep dropped
       drv dropped tx total: 0
       drv dropped rx total: 126
```

#### Host OS (ESXi)

```
/net/portsets/vSwitch1/ports/33554500/vmxnet3/> cat /net/portsets/vSwitch1/ports/33554500/vmxnet3/rxSummary
stats of a vmxnet3 vNIC rx queue {
   failed when copying into the guest buffer:0
   # of pkts dropped due to large hdrs:126
   # of pkts dropped due to max number of SG limits:0
```

Looks like it's due to large headers. I tried to hook on function which frees `skb` if error:

```
probe module("vmxnet3").function("vmxnet3_rx_error")
{
  printf("mtu: %d, fcs: %d, len: %d\n",
          $rq->buf_info[0]->len,
          $rcd->fcs,
          $rcd->len);
}
```

```
mtu: 1522, fcs: 1, len: 54
```

An interesting part is that frame checksum is OK, but packet length is 54 bytes, this is like UFO packet. Otherwise, IPv6 itself doesn't require checksums, thus this driver doesn't care about this part.

These drops appear random and cannot catch them with tcpdump/BPF, because they are dropped before in driver. To deal with this we decided to use >1500 MTU for internal traffic inside ESXi nodes.

After changing default MTU to be `1515`, problems were gone. Looks like 1515+22=1537 is enough for handling this underlying extra headers.
