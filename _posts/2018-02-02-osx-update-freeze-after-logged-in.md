---
layout: post
title: OSX fucked up again
categories:
- blog
---

As usual, the latest update broke OSX again. In most cases there are few options to fix this piece of crap:
* Reset [NVRAM](https://support.apple.com/en-us/HT204063)
* Reset [SMC](https://support.apple.com/en-us/HT201295)

But the latest update was a bit different. Output is grabbed using (Command-V)
![OSX freezes after update](/images/osx_kext.jpg)

Fortunately, there is a good source of startup key [combinations](https://support.apple.com/en-us/HT201255).

This time fix was horrible:
* Boot your OSX into [recovery mode](Command-R)
* Mount your APFS/CoreStorage (depends which version you are using of OSX)
  * `diskutil apfs list` (list volumes and copy logical volume UUID)
  * `diskutil apfs unlockVolume <logical volume UUID>`
  * `rm -rf /Volumes/Macintosh\ HD/System/Library/Extensions/*`
* Reboot

#### Conclusion

I'm just writing this to point myself or other $end_users to do not lost next time if such things happen.

I'm moving to Ubuntu.
