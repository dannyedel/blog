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

```bash
# change these
GRUB_DEFAULT=0
GRUB_HIDDEN_TIMEOUT=1 # gives 1 second window to hit ESC
GRUB_HIDDEN_TIMEOUT_QUIET=false # shows the 1..0 countdown on screen
GRUB_TIMEOUT=0 # prevents deprecation warning

# these should be default anyway
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
```

