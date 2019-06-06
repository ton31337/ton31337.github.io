---
layout: post
title: LLDP does not work with i40ge driver in VMWare ESXi
categories:
- blog
---

Imagine your services critically depend on the Link Layer Discovery Protocol (LLDP). And this is the _brownie_ if you have $VMWare ESXi and Intel 710/711722 series NICs.

LLDP just doesn't work with i40ge driver well. Well, it doesn't work at all.

Test it on the guest:

```
# lldpctl
-------------------------------------------------------------------------------
LLDP neighbors:
-------------------------------------------------------------------------------
```

The main problem is that the NIC doesn't forward LLDP packets to guests. Hence, you can't see what are your neighbors inside the guest. The bigger problem arises if you depend on LLDP and other services are failing due to that. Cascading failure scenario.

To overcome this limitation and allow guests to see LLDP packets you MUST use the newest i40ge driver and modify the flash.

Here are the commands to upgrade i40ge firmware to the newest and disable LLDP agent. When you disable LLDP agent, it will forward LLDP packets to guests:

```
# wget http://donatas.net/VMW-ESX-6.7.0-i40en-1.8.6-13636624.zip
# esxcli software vib install -v /vmfs/volumes/datastore1/__tmp/i40en-1.8.6-1OEM.670.0.0.8169922.x86_64.vib
# esxcli system module parameters list -m i40en
# esxcli system module parameters set -m i40en -p LLDP=0,0
# reboot
# wget http://donatas.net/x722-lldp.tar.gz
```
As you see from the output above, it's not enough to upgrade the firmware. We need to use the 3rd party tool which is released by Lenovo to flash the firmware and disable LLDP agent.

```
[root@us-imm-esxi7:/vmfs/volumes/5cef6df8-fabe2ba4-7e68-ac1f6b5a176e/__tmp/X722-LLDP-FW-Setting-Tool/esxin64e] ./LLD
Pn64e -disable -debug
Connection to QV driver failed - please reinstall it!
LLDP 1.0.5
Copyright (C) 2018 Intel Corporation

Scanning for matching Intel(R) devices...

## Vend Dev  MAC          Branding-String
-- ---- ---- ------------ ---------------
MAC type is 327683
 1 8086 37D2 AC1F6B5A176E Intel(R) Ethernet Connection X722 for 10GBASE-T
   FW: 3033 ETrack: 80000E48  FW:3.1 API:1.5  NVM: 3.33 MAP2.31
MAC type is 327683
 2 8086 37D2 AC1F6B5A176F Intel(R) Ethernet Connection X722 for 10GBASE-T
   FW: 3033 ETrack: 80000E48  FW:3.1 API:1.5  NVM: 3.33 MAP2.31
-- ---- ---- ------------ ---------------

2 matching adapters discovered.

About to disable LLDP in NVM on port 0
Flash size is 6004736
EMP is 8034
EMPSettingsModuleHeaderOffsetW is 0001A000
LLDPConfigurationPtr is 0007
LLDPConfigurationOffset is 0001A00D
LLDPConfigurationLength is 0008
LLDPAdminStatusOffsetW is 0001A00E
Note: LLDPAdminWord is 3333
LLDPCRCOffset is 0001A015
LLDPCRC is 8038
Changed word 0001A00E to 0000
New CRC is A4
Changed word 0001A015 to 80A4
Writing modified flash of size 6004736...done.
```

After that we have it:

```
# lldpctl
-------------------------------------------------------------------------------
LLDP neighbors:
-------------------------------------------------------------------------------
Interface:    ens192, via: LLDP, RID: 1, Time: 0 day, 01:52:28
  Chassis:
    ChassisID:    mac 80:a2:35:21:a2:71
    ...
```
#### Conclusion

This took me _four+_ hours to debug and read half the internet.
