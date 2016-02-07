---
title: Reducing backup size by using docker
tags: backup-size
---

Currently, I backup most of my Service-VMs by simply storing
their entire filesystem to the backup medium.
Add incremental backups so I don't need a crazy large supply
of backup media, but that's essentially it.
A rebuild works by creating a fresh VM, partitioning and
formatting the hard drive, populating it with the backup and
running update-grub.

These backups contain mainly binaries that could be installed
back from a distribution mirror (which may even have a faster
connection than the emergency backup site's upload), but
normally in the course of the VM's life the configuration
files of the programs got tweaked with, more or less.

This is also known as having a "snowflake" server -
it looks similar enough to one with the same packages
installed, but the amount of manual configuration involved
makes it unique.

## Reproducing servers instead of saving them

The *solution* to this problem is to
**not ever edit any configuration file or
install any software manually on the server**.
Instead, write some sort of automation that begins
with a reproducible state (base OS) and applies every
change needed to restore working operation.

Then all that is needed to backup is *this script*,
plus of course the folders where the actual user data
is stored in  (Databases, e-mails, you get the idea).

This has the added benefit that the VM becomes
*disposable* - if we bind-mount the userdata folders
from the host, we can throw away the VM's storage at
any time.  There's now a clear separation between
various tiers of data:

1. The buildscript
  * Very Important, very small, plain text.
2. The userdata
  * Very important, can be large, can be application-specific
  binary.
3. VM state generated during operation
(Think of /var/cache and similar)
  * Nice-to-have, can be large, but should persist
  between machine builds to speed up stuff
4. Bulk Data of the VM
(the installed OS, all binaries and configuration data)
  * Size enormous (gigabytes)
  * Completely reproducible from the buildscript and
  a distribution mirror

The buildscript can even be saved in *public* version control,
since it will by definition not include the userdata.

Since the machine is not meant to be manually administrated,
it will also *not* contain any passwords or publickeys.
Users' authentication credentials will be saved in tier 2.

Backup strategy:

* Tier 1: Save every little change (version control),
you will need the ability to bisect sometime in the future.
`git` is great for this.
* Tier 2: Save daily (or more often, if you cannot tolerate
10h of work lost) using a software capable of storing the data.
This may require support by the running application (think
a database that writes a consistent snapshot to disk before
you write it to backup storage)
* Tier 3: Don't backup this, but keep it between machine
builds.  Requirement:  Machine must start and work correctly
with an empty state directory, it is just allowed to be slower.
* Tier 4: Don't backup this, and throw it away on machine
rebuilds once you know the new version works.

## About docker

One system that implements this kind of method is
[Docker][dock] (package `docker.io` in Debian).
It's basically a [LXC][lxc] frontend with a lot
of bells and whistles,  so it is a
kind of *container virtualisation*.

Container virtualisation means that the VMs
*share the host kernel, especially its drivers,
memory, filesystem and process space*
instead of each having their own kernel - and it is
that host kernel's job to keep them apart from each
other.
This is similar to how the kernel protects
user A's data from user B, if user A sets her homedir
to chmod 700.
As a side note, Android uses this idea to shield apps from
each other - each app gets its own system user and the
rest is up to the kernel.

The \*BSDs have container virtualisation for years;
a container is called a `jail` there.

Upsides:

*   **Super** Fast

    There is no VM emulation overhead,
    since all the "guest" processes are native host processes

*   No large image files

    The entire guest filesystem is a directory on the host,
    removing the need for an opaque image and FS drivers
    in the guest.

*   Sharing of host directories without network overhead

    Sharing is as simple as a bind-mount.

Downsides:

*   Resource exhausting code (think forkbomb) can DoS the
    entire machine, not just the guest (in a full-system
    emulation, the guest is always only one process from
    the view of the host kernel)

*   Any kind of priviledge escalation bug will allow full
    host access.  This is a larger attack vector than just
    "a bug in the hypervisor code".

    On the other hand, "local privilege escalation" in the
Linux kernel gets patched really quick once publicly
known.  So unless you are "worthy" of your attacker
spending the black-market cost of a 0-day on you, you
can probably ignore this as long as you install security
updates immediately (this will block the script kiddies
better than any "virus scanner").

Because of this,
I'm going to ignore the downsides from now on,
since I'm not hosting top-secret stuff,
and I don't believe someone would "burn" a 0-day on me,
but rather want the advantages.

Also, I don't execute untrusted code in the VMs.
If you do, you might want to think about another solution.

If you don't believe me with the resource exhaustion thing,
run one of my [forkbombs][fb] in a VirtualBox, and then
in a docker container.  They are only a few lines of code
and have been able to bring a docker down in every one
of my attempts.

## Using docker to create reproducible machines

Docker has a couple of features that make it intresting in this
use case, versus plain LXC or OpenVZ, Jails etc.

It allows to save containers to a "repository", but also
allows building them from scripts.  These scripts reference
a "base image" (this can be a minimal Debian install, just
a busybox binary, or a complete desktop system with texlive)
and then list the commands and files to execute and place into
the base image to finish it.

You look at my [apt-cacher-ng docker script][acngdocker] to
get the basic idea.
The Dockerfile starts with a base image (debian jessie) and
from there on out copies files to the machine and executes
commands inside of it.

The "volume" of this machine is an example of a tier-3
storage.
The files serve as a cache of the Debian mirrors,
so I can install VMs with LAN-speed - but it will work
fine if this directory is empty, I just have to wait for
the cache to warm up.

## Example: Apt-cacher-ng

Once docker is installed on the host, the following commands
will run the buildscript (making an image) and then start
a container-VM from it (which will happen in *seconds* vs.
the start of a "real" VM complete with BIOS, grub, kernel,
initsystem, etc)

```bash
docker build -t dannyedel/acng --rm https://github.com/dannyedel/docker-acng.git
docker create --restart=always -m 256M -v /mnt/var_cache_apt_cacher_ng/:/var/cache/apt-cacher-ng:rw -p 3142:3142 --name acng dannyedel/acng
docker start acng
```

## Summary

Old strategy "save everything":

* On the RAID during operation
  * One VM image, probably 30G in size
* On the backup medium
  * Around 10-20G OS data + config files + cache

New strategy "docker"

* On public mirror (github)
  * Buildscript, less than 1M
* On the RAID during operation
  * Docker image, around 300M
  * One LVM volume for the cache contents, 20G
* On the backup medium
  * Only the buildscript (less than 1M)

## Other advantage

This method has the nice advantage to verify that
**your rebuild system actually works** since you're
executing it each time you are making a change to the
machine configuration.
This also means you're starting to memorize the commands
versus only looking them up when disaster recovery is
actually needed (have fun if this happens on your DNS
resolver, and you forgot to copy the docs offline)

Also, since the VMs are no longer opaque to the host,
you can run commands inside it with `docker exec`,
without needing to set up an ssh server (additional
attack vector)
or similar in the VM.
The only attack vector is the ssh of the host, but
this is also true in a full-hypervisor configuration -
as soon as an attacker gains root on the host, she
can do *everything* she wants.


[dock]: https://docs.docker.com
[lxc]: https://en.wikipedia.org/wiki/LXC
[fb]: https://github.com/dannyedel/forkbombs
[acngdocker]: https://github.com/dannyedel/docker-acng/blob/master/Dockerfile
