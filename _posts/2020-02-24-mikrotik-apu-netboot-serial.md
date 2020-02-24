---
title: Netbooting PC Engines APU into debian-installer using MikroTik
---

Everything in this article is likely applicable to other devices that
follow the basic principle of the PC Engines APU / ALIX boards.  They
have a few ethernet ports and a serial console, but no video output.
They are fully supported by a stock linux kernel (this is not the case
with a certain fruit-named board) and their on-board ROM already
contains everything needed to boot from PXE.

Note that "getting the PC Engines to execute its PXE code" is not part
of this tutorial.  Their serial-console-accessible BIOS is pretty much
self-explanatory.

## The mission

Setup a MikroTik Router to direct the APU via DHCP to the MikroTik-TFTP
server, which in turn serves a pxelinux boot file and configuration,
which then downloads the rest directly from the official debian mirror
via http.

## Configuring PXELINUX

You will need the `lpxelinux.0` file from `pxelinux` and the BIOS
version of `ldlinux.c32` from `syslinux-common`.  Upload these to a
directory of your choosing on the mikrotik, probably on an sdcard or
other flash, so you don't have to do everything again on reboot.
In this example, I will use `/rb750g-sdcard/tftpboot/` as the base
directory.

Now, write a configuration file.   See the [syslinux wiki][wiki] for
details on what you can do here.   The basic idea is that you want
to load a linux kernel and initrd from a network mirror, and add
boot options for the setup.

* `console=ttyS0,115200` is a must.  Otherwise "probing EDD" will
  likely be the last thing you see on the serial port.
* `net.ifnames=0` is a personal must for me.  See the
  [other entry in this blog][ifnames] about the reasons...
* `priority=low` means it will ask more questions.  For example,
  whether you even **want** a non-root user account, or if you
  want to leave some volume-group-space unallocated for later.
  This is also known as "expert mode".
  (I highly doubt people starting a serial console
  installation via PXE are considered "normal" users...)
* `---` (triple-dash) marks options that should be copied into the
  INSTALLED system.  You will at a minimum want the `console=`
  setting here.  Also, if you used `net.ifnames=` for the installer,
  you will want the same setting here.

Obviously, the `...` is not literal.  The [actual mirror url][mir]
is pretty long.  Also, the `APPEND` can NOT be line-broken.  Write
it all onto a single line.  Here's my starting configuration:

```text
DEFAULT debian-buster-serial

LABEL debian-buster-serial
LINUX http://ftp.de.debian.org/.../linux
APPEND initrd=http://ftp.de.debian.org/.../initrd.gz
           console=ttyS0,115200 BOOT_DEBUG=2 
           net.ifnames=0 priority=low ---
           net.ifnames=0 console=ttyS0,115200
```

Upload this file into a sub-**directory** named `pxelinux.cfg` with
the **file**name `default`.  In my own example, its on
`/rb750g-sdcard/tftpboot/pxelinux.cfg/default`.

Now, all thats left is configuring the Mikrotik-DHCP-Server to announce
those filenames.

## Configuring Mikrotik TFTP

After the files are uploaded to the mikrotik, allow access to them
in the TFTP configuration:

```text
/ip tftp add req-filename="/rb750g-sdcard/tftpboot/.*"
```

## Configuring Mikrotik DHCP-Server

Now, add the TFTP Server IP (i.e. your MikroTik's IP) and the file-path
on the TFTP server to the `network` configuration.

```text
/ip dhcp-server network
set $PUTRIGHTNUMBERHERE next-server=10.54.1.1 \
 boot-file-name="/rb750g-sdcard/tftpboot/lpxelinux.0"
```

That's it.   For debugging, you might want to add `tftp,debug` to
`/system logging` and watch the serial console output.

[wiki]: https://wiki.syslinux.org/wiki/index.php?title=Config
[mir]: http://ftp.de.debian.org/debian/dists/buster/main/installer-amd64/current/images/netboot/debian-installer/amd64/
[ifnames]: /2016/03/21/from-predictable-interface-names-to-eth0/
