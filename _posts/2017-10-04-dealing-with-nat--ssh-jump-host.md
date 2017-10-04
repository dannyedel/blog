---
title: "Dealing with NAT:  SSH Jump Hosts"
tags: nat
---

Need to reach the host behind the NAT device, that does not have a public
IPv4, and its only IPv6 connection is a makeshift tunnel?
Don't want to set up port forwarding and remember every damn
TCP-Port-To-Hostname-Mapping for each host behind the NAT?

Just ssh to the NAT device (firewall) and jump from there.

Bonus Points:  SSH has a commandline switch for that.

```bash
ssh -J regularjoe@firewall root@some-device-behind-the-firewall
```

Even works with mikrotik as a jump host, where the regular ProxyCommand
directive would fail, since it does not have `nc` installed.

-----

Edit:  This feature was added to OpenSSH in [version 7.3][1], meaning
it's available in Debian [beginning with stretch][2].

[1]: https://www.openssh.com/txt/release-7.3
[2]: https://packages.debian.org/search?keywords=openssh-client&searchon=names&suite=all&section=all
