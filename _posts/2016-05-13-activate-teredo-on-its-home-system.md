---
title: Activate teredo on its home system
---

Just for comparison, on Debian and derivates the
equivalent of this whole article is three words:

```bash
apt install miredo
```

And voila, you can access IPv6-only hosts from a
IPv4-only connection by using a public gateway.

## How the same thing works on the operating system where the protocol was invented

Call `gpedit.msc` with root permissions and navigate to
*Computer Configuration* ->
*Administrative Templates* ->
*Network* ->
*TCPIP Settings* ->
*IPv6 Transition Technologies*.

Then set

* Teredo Default Qualified: **Enabled**
* Teredo Server Name: Any (existing!) public [server][1] will do.
  If in doubt, the one configured on your OS by default is already
  turned off.
* Teredo State: **Enterprise Client**

Check `netsh interface teredo show state`.
It should say "qualified" after a few seconds.

Now you should be able to connect to IPv6-only hosts by address.

Try `ping -6 ipv6.google.com` or similar.
Then try it without hinting `-6`.
If it works, stop here and be happy.

### Part 2: Enable DNS resolution, too

If connecting to literal IPv6s or explicitly requesting IPv6 works,
but it keeps refusing to connect to IPv6 hosts by name
(even those that don't have an A record, but only an AAAA record),
then you need to convince the OS do also return AAAA responses.

The Internet™ says that needs:

* Any ipv6 address on a `Local Area Connection` network interface.
  * The site-local address `fc00::1` with a `128` netmask will work.
  * Do not set a default gateway or DNS server.
* Manually adding the default route
  * Call `route print` and check the `If` number at the `2001::/32` route
  * As root call `netsh interface ipv6 add route ::/0 interface=$THENUMBER`

Now try `ping ipv6.google.com` again.

[1]: https://en.wikipedia.org/wiki/Teredo_tunneling#Servers
