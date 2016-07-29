---
title: Fixing the 5sec delay on getaddrinfo()
---

Some broken recursive name resolvers (sadly, I have one of these) seem
to cause a timeout when looking up *both* `A` and `AAAA` records in
parallel.

The symptom is that `getaddrinfo()` always seems to take about 5 seconds
before giving an answer, making every IPv6-ready application hang at the
DNS resolve stage.  Pure IPv4 applications that use `gethostbyname()`
are not affected, neither are applications sending their own DNS packets
sequentially (such as the `host` and `dig` utilities bundled with
`bind`).

To solve this, add a line `options single-request` to
`/etc/resolv.conf`.  If you use `resolvconf` to manage this file, the
line should go into `/etc/resolvconf/resolv.conf.d/tail`, this will
make `resolvconf` write it at the end of `/etc/resolv.conf`.
