---
---

If computers act strange, it could be a case of bad memory.
Often, not the entire module is defective, but a specific region of it.
With memtest86+, grub2 and (of course) Linux we can configure the system to
	never use these defective regions, extending the life of an old computer
	by a few months or even years.

## Step 1: Find defective regions with memtest86+

Simply load it from any Live CD.
For this experiment I used a Lubuntu 14.04 i386 disc
	which contains Memtest86+ 4.20.
The memtest86+ code contains instructions to activate long mode when run on
	an x86-64 processor, so the same image works for 32bit and 64bit PCs.

After it loads, execute the following commands

* `c` enter configuration mode
* `4` Error Report Mode
* `6` Beep on Error (so you don't have to look at the machine all the time)
* `4` Error Report Mode
  (Take a quick look, Beep on Error **should be checked now.**)
* `3` BadRAM Patterns
* `0` continue

Now get coffee.
This literally takes hours on each of my systems.
Hope it doesn't start beeping.


### Result 1: All good.

If memtest reports "Pass complete, no errors" simply keep it running for a few passes extra,
to make sure everything is still good when the components get warm.
If it still doesn't show any problems, be happy and leave it be.

### Result 2: Problems

Write down the BadRAM patterns, best to a piece of paper that gets stored with
	the computer in question.
Just in case it ever needs to be reinstalled, it's always good to be reminded of the
fact it has defective RAM.

## Step 2: Reinstall OS

***Anything*** that was on the system is most likely utterly broken.
Whatever data was loaded into the RAM and written back to disk is **corrupt**.
Good thing that all your important data is not only on this harddisk,
	but on some external *and verified* backup or in distributed version control,
	so it's a simple case of re-installing the system.

Since we now know which memory regions are defective, we can tell the Linux kernel
(of the live/installation CD) to not use them.

The kernel boot param is `memmap=0xSIZE$0xSTARTLOCATION`.

Suppose the badram bitmask is `0x0c406b6c,0xfffffffc` ,
	we could simply round up to `0x10` size and do a
	`memmap=0x10$0x0c406b60`, effectively blocking
	`0x0c406b60`…`0x0c406b6f`, which includes all the defective bits.

Just remember to place the `memmap` parameter **before** the `--` if that is included
	in the kernel param, since everything after it will get passed to `init` and ignored
	by the kernel.

Once the Live-CD is loaded, take a look at `dmesg`.
The first few lines should reflect that the faulty memory has been marked as reserved.

**Now perform the normal OS installation.**
The kernel will not use the "reserved" memory.

When the installation is finished,
	***do not reboot*** into the installed OS;
	the freshly installed kernel doesn't know about the defective RAM yet.


## Step 3: Tell new grub/Linux combo about the defective RAM

GRUB2 can be fed a BadRAM pattern, which it will write to the `e820` BIOS map,
	which gets honored *both* by the Linux Kernel and memtest86+.


```bash
mount /dev/newmachine-vg/root /target
mount -t proc proc /target/proc
mount -t devtmpfs dev /target/dev
mount -t sysfs sysfs /target/sys
chroot /target /bin/bash
mount /boot
```

Now edit `/etc/default/grub` and put the BadRAM pattern into the
	`GRUB_BADRAM` variable, exactly as output by memtest.

Run `update-grub`.

If it is not yet included (ubuntu includes by default),
	run `apt-get install memtest86+`.

```bash
umount /boot
exit # leaves chroot
umount /target/{proc,sys,dev,}
```

## Step 4: Verify
Reboot the system, make sure to boot into the `memtest64+` just installed,
	**not** the actual system.

Let it run through, it should **not** detect **any** memory errors, because
	memtest will respect the `e820` memory map marking the defective parts.

If it still finds any errors, restart at the beginning, since the installed
	OS is most likely corrupt.
Because of the way the linux buffer and cache work,
	it's virtually guaranteed that the defective memory got used at
	some point during OS installation.
