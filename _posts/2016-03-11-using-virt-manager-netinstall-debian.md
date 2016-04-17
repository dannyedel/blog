---
title: Using virt-manager's netinstall feature for Debian
tags: smaller-backups
---

Installing a debian VM using virt-install's (integrated into virt-manager)
netboot feature saves me from keeping yet another ISO file on the harddisk,
which would otherwise taking up space and bitrot away.
But I always forget the correct path to pass...

Install from de mirror:

```
http://ftp.de.debian.org/debian/dists/stable/main/installer-i386/
http://ftp2.de.debian.org/debian/dists/jessie/main/installer-amd64/
```

Install from my local apt-cacher-ng:

```
http://acng:3142/debian/dists/stable/main/installer-i386/
```


On "kernel options", don't forget to specify

```
priority=low
```

in order to be asked all the questions (equivalent to *expert mode* on the ISO).

Alternatively,

```
auto=true priority=critical url=autoinstall.server.simon-mueller.de
```

for our basic VM provisioning.


-----

Updates 2016-04-17:

* Add ftp2 (ipv6 enabled) mirror with amd64
* Add string for VM provisioning
