---
title: Going from (un)predictable interface names back to eth0
---

Interfaces names that do not change across reboots are a blessing for
those with multiple ethernet cards connected to different networks,
routers for example.
However, there is a downside:

When moving harddisks between (virtual) machines, the MAC addresses of
the network cards change, so the first interface is no longer eth0 but
eth1.  Previously, deleting `/etc/udev/rules.d/70-persistent-net.rules`
and rebooting helped.

With [predictable network interface names][1] the device name is derived
from the hardware.  This is predictable in the sense that, instead of
"the order the modules were loaded on first boot", which is more or less
random, we can tell from a `lspci` which names they will have.
This also does not need write access to the root partition on first boot,
which is essential to boot from ROM.
Booting from ROM is a security heaven (if storage is not writable, attacks
cannot persist across a power cycle), and also necessary for some embedded
devices that don't even have read-write storage.

However, this makes "The name of the first ethernet interface" **completely
unpredictable** unless you happen to memorize all of your PC's hardware.
I don't.

So, long story short, how do I get rid of it?

Append `net.ifnames=0` to the kernel command line.
That's it.
No more deleting files after reboots or moving the harddisk to a different PC.


[1]: https://www.freedesktop.org/wiki/Software/systemd/PredictableNetworkInterfaceNames/
