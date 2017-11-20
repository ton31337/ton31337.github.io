---
layout: post
title: Carry MAC address with the BGP announcement
categories:
- blog
---

Suppose we have such a local FDB table and wanna to redistribute it via BGP to remote endpoints. We encapsulate this metadata information directly in large BGP community which if large enough to carry all the necessary data - MAC, VLAN, VNI.

```
fdb = {
  '1.1.1.1' => {
    'mac' => '0a:00:27:00:00:0b',
    'vlan' => 4
  },
  '2a02:4780:bad:c0de::bad' => {
    'mac' => '0a0027-000005',
    'vlan' => 5
  }
}
```

Encapsulate by splitting MAC address into two parts because MAC address is 48-bits and BGP large community allows you to use three 32-bit length spaces.

Hence, we can put first 24-bits into the first 32-bit space, another one to the second, while third will be left for the VLAN identifier.

```
+--------------------------------+
| MAC first (24-bits) | (8-bits) |
+--------------------------------+
| MAC last (24-bits)  | (8-bits) |
+--------------------------------+
| Metadata: VLAN, VNI  (32-bits) |
+--------------------------------+
```

Pseudo-code looks like below:

```
def community(meta)
  mac_hex = format('0x%s', meta['mac'].gsub(/[:-]/, '')).hex
  low_order_bits = mac_hex & 0xffffff
  high_order_bits = mac_hex >> 24
  format('%d:%d:%d', high_order_bits, low_order_bits, meta['vlan'])
end
```

When we have encoded MAC address for a given IP address we can send this community along with IP address hooked up. To decapsulate from a custom format back to original MAC address we can use this pseudo-code:

```
def restore_mac(*args)
  args.map do |part|
    part.convert_base(10, 16).rjust(6, '0')
  end.join
end
```

#### Conclusion

Why this is needed? Maybe it could be reused in such cases like EVPN istead of creating another one NLRI?

>A complex system that works is invariably found to have evolved from a simple system that worked. A complex system designed from scratch never works and cannot be patched up to make it work. You have to start over with a working simple system. – John Gall (1975, p.71)
