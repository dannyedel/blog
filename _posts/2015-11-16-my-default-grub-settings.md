---
---

I don't like Ubuntu's default grub settings because they hide the
grub screen completely.  While the idea of booting as fast as possible
is generally okay, sometimes I need to enter the bootloader, and the
default hides the countdown to tell me when.

## Desktop/Laptop system (Ubuntu, end-user)

* Optimized for boot time (only 1 second pause)
* Shows Ubuntu splash screen
* No deprecated configuration warnings

`/etc/default/grub`:

```bash
# change these
GRUB_DEFAULT=0
GRUB_HIDDEN_TIMEOUT=1 # gives 1 second window to hit ESC
GRUB_HIDDEN_TIMEOUT_QUIET=false # shows the 1..0 countdown on screen
GRUB_TIMEOUT=0 # prevents deprecation warning

# these should be default anyway
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash net.ifnames=0"
GRUB_CMDLINE_LINUX=""
```

run `update-grub` and reboot

## Workstation (Debian, poweruser)

* Doesn't get rebooted often, 5 secs wait is okay
* I like seeing messages so I know I havn't broken it yet
* Adds kernel parameters to enable Docker's memory and swap limiting

```bash
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1 net.ifnames=0 verbose"
```

-----
edit 2016-03-21: Appended `net.ifnames=0`.
