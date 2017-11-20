---
title: DNSSEC with unbound - fast, but verified
---

For speed reasons, many use a publicly available resolver such as
google's public DNS or their provider's DNS.

This is all fine, and some of them even announce that they provide
DNSSEC validated outputs.  But if we're assuming that some attacker is
trying to supply us with crafted DNS packets, what's to stop them from
simply altering the packets from google to us?

Answer: Unbound.

## Installation and configuration on Debian (tested on stretch)

Execute the usual incantation as root:

```bash
apt install unbound
```

That's it.  You now have:

* Your local clients use unbound instance for lookups
  * Check `/etc/resolv.conf` --
  the only namserver in there should be `127.0.0.1`
* Unbound's default configuration is made for this:
  * It listens on localhost only, so other hosts on your network cant
  (ab-)use you for IPoverDNS (yes, that's a thing.)
  * Unbound will resolve and verify DNS

There's only one downside:  This is all **a lot slower** than before.

Your client walks through the entire recursion itself, connecting to
servers which can be very far away, spending multiple seconds on network
roundtrips.

## Speeding things up

We used google public DNS for a reason -- it's fast.

So let's direct the new unbound instance to the google farm for speed,
while still verifying their results locally, to be protected from
DNS-based spoofing.

Drop a config file into `/etc/unbound/unbound.conf.d/`.  For example,
here's my `/etc/unbound/unbound.conf.d/speed-and-validate.conf`:

```text
server:
	prefetch: yes
	serve-expired: yes
	verbosity: 1
	harden-below-nxdomain: yes
	val-log-level: 2

# Speed up by using google public dns
forward-zone:
	name: "."
	forward-addr: 9.9.9.9  # Quad9
	forward-addr: 8.8.8.8  # Google A
	forward-addr: 8.8.4.4  # Google B
```

### Explanations of the variables:

* `prefetch` will issue queries when clients ask for stuff that is
  *still* in the cache, but low on remaining TTL (10% of original TTL).
  This way, the client instantly gets an answer (from cache), but the
  server already refreshes the data.
* `serve-expired` unbound will give out its expired cache entries with
  zero TTL to the client while looking up in the background.
* `verbosity` Change to your personal liking.  On level 2 you'll see the
  queries and which upstream server they'll be sent to.
* `harden-***` see manpage
* `val-log-level: 2` shows validation errors.

The `forward-zone` clause redirects everything to those servers.  Feel
free to add more fast servers / server networks.

## Other new problems:  Local fake DNS zones

While this is technically a feature, a local fake DNS zone (very popular
is `<hostname>.fritz.box.`) does not work anymore.

If you want to exclude this zone from DNSSEC and forward it to your
router, add another config file.

Here's my `/etc/unbound/unbound.conf.d/fritzbox.conf`:

```text
server:
	local-zone: "45.46.10.in-addr.arpa." transparent
	domain-insecure: "fritz.box."

forward-zone:
	name: "45.46.10.in-addr.arpa."
	forward-addr: 10.46.45.1

forward-zone:
	name: "fritz.box."
	forward-addr: 10.46.45.1

forward-zone:
	name: "2.0.0.2.ip6.arpa."
	forward-addr: 10.46.45.1
```

## Remote control

To control unbound at runtime (i.e. change the servers used for
forwarding) you can activate the remote control.

During installation, unbound created a public/private keypair in
the `/etc/unbound` folder, which is only readable by root.

So activate remoting in another config file, here's my
`/etc/unbound/unbound.conf.d/remote-control.conf`:

```text
remote-control:
	control-enable: yes
```

Now you can run, for example,

```bash
sudo unbound-control forward
```

to list the current forward DNS server(s), and you can run

```bash
sudo unbound-control forward 10.46.45.1
```

to overwrite them with the new (possibly DHCP-learned) only host that
should be used.

If you want to disable forwarding, and query the official servers
directly, issue

```bash
sudo unbound-control forward off
```

Restoring the written configuration can be achieved with

```bash
sudo unbound-control reload
```

but be warned that this empties the cache.  If you only want to restore
the forwarders, just add them again:

```bash
sudo unbound-control forward 9.9.9.9 8.8.8.8 8.8.4.4
```
