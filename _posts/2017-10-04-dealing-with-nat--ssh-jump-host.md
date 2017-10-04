---
title: "Dealing with NAT:  SSH Jump Hosts"
tags: nat
---

Need to reach the host behind the NAT device, that does not have a public
IPv4, and it's IPv6 connection is only a makeshift tunnel?
Don't want to set up port forwarding and remember every damn
TCP-Port-To-Hostname-Mapping?

Just ssh to the NAT device (firewall) and jump from there.

Bonus Points:  SSH has a commandline switch for that.

```bash
ssh -J regularjoe@firewall root@some-device-behind-the-firewall
```

Even works with mikrotik as a jump host, where the regular ProxyCommand
directive would fail, since it does not have `nc` installed.
