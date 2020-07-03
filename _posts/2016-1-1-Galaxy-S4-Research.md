---
layout: post
title: Samsung Galaxy S4 Research
---

## Heads up
This isn't going to be formatted as a white-paper, and is instead, formatted as a narrative, as there's been a large request for such.

This research was by myself (Nolen Johnson), and Ryan Grachek.

With awesome help from several other people along the way.

## Glossary
- Bootloader: A general term for a link in the boot-chain that has a specific job that is run each cold-boot
- CAF: The Code-Aurora Foundation, Qualcomm's open source release entity.
- cold-boot: Fresh boot from powered off state
- QFUSE: Microscopic hardware fuse that is integrated into the SoC - Once physically blown, impossible to reset or replace
- SoC: System-on-chip (your phone’s “motherboard” of sorts)
- EFUSE: Software based fuse whose data is stored in QFPROM
- QFPROM: Qualcomm’s fuse region
-  TrustZone: Qualcomm ARM chipset’s “Secure World” implementation
- QSEECOM: A linux kernel driver that allows communication with TrustZone, and the issuing signed and verified SCM calls to do things like blow fuses. Please note that only signed applets and OEM/ODM approved calls can be made
- SCM: Secure Channel Manager (note: not related to Linux’s SMC calls)
- SMC: The Linux Kernel's "Secure Monitor Call", read more [here](http://infocenter.arm.com/help/topic/com.arm.doc.den0028b/ARM_DEN0028B_SMC_Calling_Convention.pdf)
- DTB: Device Tree Blob. Its purpose is to “provide a way to describe non-discoverable hardware” to Linux, read more [here](https://elinux.org/Device_Tree_Reference)
  
## Problem Statement
Starting with the Galaxy S4, Samsung decided to implement their new "KNOX" security mechanism(s), and no longer allow end-users to unlock the bootloader on their devices. Though the modification of personal wholly-owned devices was proven to be entirely legal back in 2010 (see [here](https://washtechu.org/index.php/sitelink/blog/65-the-ethics-of-jailbreaking) for more information on Jay Freeman's amazing legal crusade on this), it was _not_ ruled that all devices must be modifiable. This means that device manufacturers are legally allowed to place digital signature verifications in place that prevent running your own operating system, or even, in Apple's case, your own applications.

On top of the above, Samsung's TouchWiz skin from this era was _laggy_ under load, and I was limited in the monetary department. I picked up my friend's Google Play Edition Galaxy S4, and was in _awe_ at how buttery it was in comparison. I figured converting one variant to another couldn't be _that hard_, and I can't state enough how wrong I was.

## Idea: Variant Conversion

This concept was dropped quickly.

The Google Play Edition's images are signed with a different RSA key than the standard versions' firmware. Additionally, the device's PBL won't boot an SBL1 that isn't intended for your specific variant, so that makes it impossible to convert fully.

We may still be able to run the system image from this variant, as it is unverified, but this would prove challenging, and I'll get back to this later.

## Changing the Goal

If we can't run legitimate Google Play Edition images, LineageOS is a solid alternative, it's an OSS distribution of Android that can be easily ported to new devices in most cases. What's even better? They already support the SCH-I545! Apparently there are "Developer Edition" devices that can be purchased that have factory unlocked bootloaders, and some early firmware revisions of the typical end-user model are vulnerable to Dan Rosenberg's [Loki](http://blog.azimuthsecurity.com/2013/05/exploiting-samsung-galaxy-s4-secure-boot.html), which uses the fact that ramdisk loading addresses aren't sanity checked before the unverified ramdisk is loaded into memory. Dan used this vulnerability to overwrite the applications-bootloader in memory with shellcode to nullify RSA signature checks.

Well, we could just downgrade to a vulnerable firmware, right? No, sadly Qualcomm's QFUSEs are put to great use here, and the row known as `SW_REV` is up-revisioned upon most major updates, and my device was on a much newer rollback revision.

I suppose our goal has now changed - we can either:

 - Unlock the bootloader fully
 - Make the device _think_ it is a developer edition

## What are we working with?

For a better understanding of the chain-of-trust on Qualcomm's apq8064 platform, see the relevant section on my writeup on the matter,  [here](https://lineageos.org/engineering/Qualcomm-Firmware/).

[![](https://lineageos.org/images/engineering/content_qualcomm_firmware_0.png)](https://lineageos.org/images/engineering/content_qualcomm_firmware_0.png)

  
## Digging in
  
I began my research by dropping an SCH-I545's VRUFNK1 (my current firmware) aboot.mbn into IDA, only to discover that, as I had somewhat expected, it had it's symbols stripped. I then proceeded to grep the bootloader for a relevant git tag to give me some clue as what revision of Qualcomm's LittleKernel this was based on. After finding the closest match, I built it for the apq8064 platform, and loaded it into IDA with symbols intact so that I could match up all possible symbols and functions.

After a fair amount of research, I found a few interesting things worth noting:

- A QFUSE labeled `TESTBIT` (at row `0xfc4b80ac`) which if enabled controls whether the local `SYS_REV` (SECURE VERSION) is compared against when booting the device
- The Developer Edition of the device could potentially be used to unlock production models, but it's complex.

### TESTBIT
In late 2014, the Verizon Note 3 had an important, and revealing leak. The firmware VRUFNC2 leaked, which was an early, internal KitKat build, with _drum roll_ full symbols! I decided to apply what I knew to this new firmware, and try to later back-port my findings to the Galaxy S4 family. Thankfully, they're very similar in terms of locking mechanisms.

Before I get into it, let me take a second to give Dan Rosenberg a shoutout here for helping me get my foot in the door, and answering questions along the way here.

The function that checks `TESTBIT` (relevant QFUSE row @ `0xfc4b80ac`) against the bit-mask for `SYS_REV` (@ `0x80030`) is `sub_F80306C`, the function is called from two places, `sub_F821514`, which doesn't appear to do anything cool, beyond providing some diagnostic output about various security settings, and in `sub_F808704`, which is _far_ more interesting.

`sub_F808704` appears to check local `SYS_REV` against the installed firmware's value. Okay, but one assume that the logic would dictate that the fuse row be blown to a value that would prevent malicious hackers from disabling rollback, right? Well, not in this case, if we take `TESTBIT` and mask it to `0x80030`, the device will, in essence, skip rollback verification, jackpot!

It's worth noting at this point that there is an additional call to `sub_F818FFC` when flashing firmware via Samsung's ODIN flash software, which gets the relevant `SYS_REV` value at flash-time, and ODIN will fail with "SW REV CHECK FAIL : Fused XXX > Binary XXX". This cannot (as far as I can see) be bypassed, but using the good ole' [Futex vulnerability](https://www.appdome.com/blog/the-futex-vulnerability/) we can gain root quickly on the aforementioned VRUFNK1 firmware, and then use the GNU tool `dd` to flash the images and bypass this.

I then hunted `TESTBIT` down on the aforementioned Galaxy S4 firmware, and some logical flows from IDA can be seen below:
[![](https://2.bp.blogspot.com/-SxEVgG-TksQ/VU-fLAcD_cI/AAAAAAAAMLQ/ofY-JujVefU/s320/testbit1.jpg)](http://2.bp.blogspot.com/-SxEVgG-TksQ/VU-fLAcD_cI/AAAAAAAAMLQ/ofY-JujVefU/s1600/testbit1.jpg)
[![](https://4.bp.blogspot.com/-N-LH3X6QFgc/VU-fM32nnaI/AAAAAAAAMLY/jXOytvzSjro/s1600/testbt2.jpg)](http://4.bp.blogspot.com/-N-LH3X6QFgc/VU-fM32nnaI/AAAAAAAAMLY/jXOytvzSjro/s1600/testbt2.jpg)

Using the above, we can also deduce that the value of `TESTBIT`’s shadow in memory is `0x700438`+`0x4000`. 

With this info, we /should/ be able to use JTAG on the bare-board from a Galaxy S4 to use hardware interrupts, hand-set the value and boot an MDK about, meaning we can use Dan's Loki exploit to boot custom firmware! This isn't a full solution, as JTAG isn't feasible in every-day usage, but as a PoC, sure!

The only way to permanently blow the fuse would be to use a flaw/vulnerability in TrustZone blow the base fuse to our desired value. Dan's QSEE exploit could easily be leveraged. We could even likely apply [M0nk's work](https://github.com/monk-dot/DefusingTheDragon) on msm8974 to run shellcode as TZ to blow the fuse.

However, I had missed something, it became clear that this wasn't feasible, as on the Note 3 at least, I had missed that this logic was in the application's bootloader _itself_, and that the lower level bootloader, SBL1, never checked that fuse, and therefore, this fuse could be used to boot older recovery/boot images, but not aboot images, ugh.

### Converting to a Developer Edition?
On boot, the last bootloader of the SBL chain reads out the eMMC CID on boot, and compares it against an optionally present hash of the CID in the footer of the aboot partition of the device to determine whether or not the device is a Developer Edition, and by proxy if it should be unlocked. Unfortunately, on Toshiba eMMC's (which the S4 has), there are no vendor commands to write the device's CID. On the Galaxy S5, which uses Samsung eMMC's, this concept was later leveraged into [SamDunk](http://theroot.ninja/disclosures/SAMDUNK_1.0-03262016.pdf) by Sean Beapure, which ultimately unlocked bootloader of most of the Samsung msm8974 family, doh!

## Moving on

What if we _didn't_ exploit the applications-bootloader, what if instead we just overwrite the kernel in memory to run a custom kernel? Enter `kexec`.

`kexec` is a tool used to overwrite the Linux kernel in memory with a new kernel, and then execute the new kernel without ever taking the system down for full reboot (hardware always remains live). It has become increasingly more common to see `kexec` implementations to bypass android device Bootloader security, as many times, it is far simpler than finding a vulnerability that would allow a full unlock.

`kexec` must be loaded in as a kernel module to work correctly. So, to even attempt this method bring up, we first need root access and the ability to load custom (un-signed) kernel modules. I will be speaking about the Verizon Galaxy S4 almost exclusively here, although this theoretically applies to most Samsung devices.

As described above, we already have root-access. Next we would need to be able to load unsigned kernel modules. In the Linux kernel, the process that checks to see if kernel module signatures are require is called `LKMS`. There is a known exploit by XDA forum user Jeboo that was initially created for Android 4.3 JellyBean called BypassLKMS, that was later ported to Android 4.4.2 KitKat by XDA forum user Surge1223.

I then unpacked the VRUFNK1 boot image, and decompressed the zImage, then loaded it into IDA, and traced the function back to find the relevant addresses in memory.
  
I then proceeded to script it up into a script I called `modload` and ran it, as you can see below:![](https://i.imgur.com/Yk9uq6e.png)
  
Now, we build the `kexec` kernel module in the Samsung provided NK1 kernel source.

Now to actually execute it! We then boot up, unmount everything we can, push the built relevant modules, custom ramdisk, and custom kernel to the root of the device, connect the device to your computer via USB, and run commands with similar syntax(s) to these:
`adb shell
su # Get root
insmod kexec_arm.ko && insmod kexec_machine.ko && insmod kexec.ko # Load our kernel modules
kexec -l /stock_nk1_zImage --ramdisk=/ramdisk.cpio.gz --append="($cat $cmdline)" --atags-file=/atags # Load the kerenl into memory
unmount –a # Unmount all possible partitions
kexec- e # Execute the kernel`

  Protip: Run `adb shell su –c “dmesg”` in another terminal to debug the process.
  
The system then attempted to soft-boot the kernel. It then failed due to a Samsung implemented mitigation to this, Samsung TIMA. I spent about 2 months debugging the various hangups I hit, and ultimately gave up.

## Accepting semi-success?
I ended up spending weeks reverse-engineering various components from the stock system, and trying to adapt the Google Play Edition's system image to work on our variants kernel. Ultimately, this worked pretty well, small things were broken, but I lived with it for a _lot_ longer that I should have.
  

## Nah, Backstreet's Back Again
I finally came back to this project after a few months, and ultimately discovered that an old vulnerability that I had missed actually applied to this device! In a Windows Insider leak, a bug in SBL1 was described wherein PBL doesn't load SBL1 by partition name, and instead loads it by block location. We are therefore able to craft a custom GPT (partition table) that shaves off a small portion of SBL1, and labels it as a new partition called `hack`, we can then write ARM32 shellcode to that partition, and PBL jumps to it. From there we can do... well, anything.

As a fun note, we used this to shine a spotlight on a friend of ours, whose face (with his permission), we packed into the ODIN flash mode in place of the typical Android-mascot logo...
![](https://i.imgur.com/Jx1uyOr.jpg)

Alongside this, I then worked with others to get the S4 booting a modern Android version, and am maintaining LineageOS builds for all Galaxy S4 variants, you can find the builds, and installation instructions, [here:](https://wiki.lineageos.org/devices/jfltevzw/)

Now my Verizon Galaxy S4 is running Android Q flawlessly!
![](https://i.imgur.com/YIBG45u.jpg)
