---
title: Quick IPv6 Access from an IPv4-only network
---

If you are in an IPv4-only network
(public WiFi for example)
and you need to access an IPv6-only service
there is a few things that should™ work.

## Why bother setting up IPv6 at all?

Simply put, because the addresses are *long enough* so that
every host (and not just every router) gets *a unique address*.

Also, unlike the common IPv4-behind-NAT setup, both hosts
**know their own address** so they can initiate a direct
client-to-client connection without the need for triangle
routing.  This is very useful for remote support setups.

## HTTP: SixXS's public HTTP proxy

SixXS operates a [public IPv4 -> IPv6 proxy][proxy]
which responds to addresses of the form
`http://[target-site].sixxs.org`.

* Example: <http://www.kame.net.sixxs.org/>
* Use policy: <https://www.sixxs.net/tools/gateway/aup/>

Downsides
: 1. Only works with unencrypted HTTP client connections,
and will change the site being loaded out of necessity.
For example, it must rewrite requests to `images.site.tld`
to `images.site.tld.sixxs.org` to preserve site functionality.
1. Does not give a publicly routable IPv6 address, therefore
not allowing incoming connections.
1. HTTP connections only,
so not usable to access IMAP or SSH.
1. Unencrypted only,
so not usable to transmit sensitive data,
such as passwords.

Upsides
: 1. Absolutely no client-side configuration required --
if you can access IPv4 HTTP servers, you can use SixXS's proxy.
1. All traffic is target port 80,
which is allowed in almost any public WLAN.
1. This should™ even be compatible with networks
that run transparent proxies, since all content
is *actually* HTTP,
so the proxy can inspect it and will most likely not block it.

[proxy]: https://www.sixxs.net/tools/gateway/

## Teredo

[Teredo][] is a transition mechanism which allows an IPv4-only
host to communicate with the IPv6 network.

Downside
: Requires IPv4/UDP access to the teredo gateway
and UDP NAT traversal,
which may be blocked in some public or university networks.

Upsides
: 1. Full IPv6 connectivity -- the teredo client gets a publicly
routable IPv6 address allowing for direct connections.
1. No restrictions on the IPv6 traffic, allowing to access
SSH, IMAPS, HTTPS etc.
1. Requires no assistance from the local network administrator
other than *not explicitly blocking* UDP --
from the router's point of view, all traffic is IPv4/UDP.
1. Designed to work behind IPv4 NAT (does not need a publicly
routable IPv4 address or port forwardings).

Installation and configuration on Debian(-ish) systems
: * `apt-get install miredo`
* This will create a `teredo` interface with a low-priority
(metric 1029) default route, ensuring that a "real" IPv6 route
gets preferred whenever connected to an IPv6 network,
so this may be a smart package to keep installed on a notebook.

Testing teredo
: If you are on a native IPv6 network, and want to use
teredo instead of your native IPv6, disable IPv6 (temporarily)
on your real interface with
`sysctl net.ipv6.conf.eth0.disable_ipv6=1`
to force outgoing IPv6 connections to go via teredo.

[Teredo]: https://en.wikipedia.org/wiki/Teredo_tunneling
