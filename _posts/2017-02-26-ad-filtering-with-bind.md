---
title: Filtering advertisements with bind9
---

I generally don't mind viewing advertisements (ads) in webpages,
in exchange for receiving content and/or services without payment
directly from me.
But some advertisers really overdo it --
especially in terms of CPU usage, resulting in my machines' web
browser's UI freezing just to render their flashy awesome banners,
custom fonts, 30-40 javascript programs, etc. etc. etc…

The good news is that these ads are mostly served from some set of
dedicated hostnames, and not from the hostnames I actually want to visit --
most ad blocker plugins use the same basic principle to block ads,
by filtering all browser requests through a list of known-bad hostnames
or filtering full URL patterns.
Sadly, this has to be done on the target computer,
often in javascript or some other language allowed for browser plugin
programming.

Filtering based on the hostname can however be done network side,
allowing significant improvements in terms of speed by using software
specifically designed to work with hostnames
(in this case, resolving them to IP addresses)
and then abusing that software.

Basically, I make my local recursive bind server pretend to be the
master nameserver for the ad domains and serve an empty zone file,
resulting in it responding with NXDOMAIN to anything within the domain.

Recipe for this null zone file:

```bind
$TTL 1d

@	IN SOA	ns0.nulldomain.invalid.  hostmaster.nulldomain.invalid. (
		1  ; serial
		8h ; refresh
		2h ; retry
		1d ; expire
		1d ; negative ttl
		)
	IN NS ns0.nulldomain.invalid.
	; no records of any other kind
```

Feel free to alter this null zone file to your liking, but since it is
supposed to be a no-op, there's not much point.
The only things technically required for bind to load it are the SOA and
one NS record.
I highly recommend using some obviously wrong TLD, like in this case
`.invalid`, to avoid any confusion.

Next, put this zone file to good use.  In your `named.conf.local` (for
example):

```named

options{
	# If you don't intend to give an absolute path to your null zone
	# file, place it inside this directory
	directory "/your/bind/directory";
	…
}
…
zone "please-block-me.com" { type master; file "null.zone.file"; notify no; };
```

Let's give it a test drive:

```bash
dig please-block-me.com
…
;; QUESTION SECTION:
;please-block-me.com.		IN	A

;; AUTHORITY SECTION:
please-block-me.com.	86400	IN	SOA	ns0.nulldomain.invalid. hostmaster.nulldomain.invalid. 3 28800 7200 86400 86400
…
```

Success!
The nameserver answers that this hostname has no IP address
associated with it.
A browser will therefore *not* try to load any data
from this server, and waste only minimal CPU cycles.
As an added bonus, the negative TTL of 1 day allows the client's stub
resolver to cache this fact for quite some time,
even removing further roundtrips between the name server and the client.

-----

Now, all we need is a list of known ad server hosts in some format we
can convert to a bind configuration file.

Luckily, you can find such lists "en masse" on the internet.

-----

COMING SOON:
Some regex magic to convert certain lists formats to bind config,
including automation for updating.
