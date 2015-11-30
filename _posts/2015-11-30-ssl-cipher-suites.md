---
title: SSL Cipher Suites
---

There are many lists around the internet of what "good" ciphersuites you should
configure on your webserver, but (most of) those lists seem to ignore one fact:
**OpenSSL itself already curates such a list** and thus we can use it instead of
manually specifying ciphersuites.

*Quick check*:
If the "awesome list from the internet"
contains things like `!NULL` or `!RC4`,
you will most likely have to keep track of which ciphers
have been broken recently.

## tl,dr: For a reasonable browser support

* Use 2048 bit DH primes (not enough support 4096 yet)
* Disable SSLv2 and SSLv3 (only allow `TLSv1.0`, `TLSv1.1`, `TLSv1.2`)
* Make sure to **use the server preferred order**,
check your servers documentation for the exact config file name.
* Set the ciphers to `HIGH+EECDH:HIGH+EDH:@STRENGTH`
  * Only forward secrecy cipher suites
  * Prefers EECDH (normally faster) over EDH for same key length
  * otherwise prefers higher key length
* This will support all ssllabs test browsers except:
  * IE6/WinXP (only SSLv2/3)
  * IE8/WinXP (no FS)
  * Java 6u45 (max 1024bit DH primes)

## The OpenSSL HIGH list

OpenSSL includes a list of `HIGH` ciphers.
It is safe to assume that a "broken" cipher,
meaning one with a known *efficient* attack,
will be removed quickly from the list.

If we want to secure a webserver,
it is important that we use *forward secrecy*,
this means that the communication is encrypted
using temporary per-session keys that
do not get stored on the server.

Basic advantage:
If the key gets factored
(remember, RSA keys are essentially a product of two large primes)
the *past communication* cannot be read,
it is only possible to impersonate the server
from there on out.
So we hope this happens *after* the certificate expires,
and we generate a new one using a new key.

You can basically build an OpenSSL
*set of ciphers* using the operators

* `+` for [intersection]
* `:` for [union]

Plus, you can use `@STRENGTH` as a pseudo-set (behind a `:`)
to sort the current set by key length.

Useful pre-defined sets (with links to wikipedia)

* `HIGH` (stream ciphers)
  * Includes only 128-bit and higher ciphers
  * Currently also includes 112-bit 3DES (but not regular 56-bit DES)
* `EECDH` (key exchange)
  * [Ephemeral][eph] [Elliptic Curve][ec] [Diffie-Hellman][dh] key exchange
  * Forward secrecy, fast, relatively new, good support for current browsers.
* `EDH` (key exchange)
  * [Ephemeral][eph] [Diffie-Hellman][dh] key exchange
  * Forward secrecy, not so fast, but been around longer, so its a good fallback.
* `RSA` (key exchange)
  * [RSA][rsa] key exchange (using the server's *long-term private key directly*,
  as in **no forward secrecy**)

[intersection]: https://en.wikipedia.org/wiki/Intersection_%28set_theory%29
[union]: https://en.wikipedia.org/wiki/Union_%28set_theory%29
[eph]: https://en.wikipedia.org/wiki/Ephemeral_key
[ec]: https://en.wikipedia.org/wiki/Elliptic_curve_cryptography
[dh]: https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange
[rsa]: https://en.wikipedia.org/wiki/RSA_%28cryptosystem%29

Note that the *key exchange* related sets (`EECDH`, `EDH`, `RSA`) do
not at all restrict the *cipher* used after key exchange
(they even contain the NULL cipher or the broken RC4)
just as the *cipher* set (`HIGH`) does not restrict the key exchange
(it allows Anonymous key exchanges, for example)
so it ***is very important to intersect them*** before using.

Normally we want to **enforce forward-secrecy** but at the minimal
possible performance cost.  Right now, that means preferring `EECDH`
over `EDH` when the browser supports it.

So we use the set `EECDH+HIGH:EDH+HIGH`.
Since the sets are already ordered by key length,
this currently gives the following priority list:

1. EECDH 256 bits AES/Camellia
1. EECDH 128 bits AES/Camellia
1. EECDH 112 bits 3DES
1. EDH 256 bits AES/Camellia
1. EDH 128 bits AES/Camellia
1. EDH 112 bits 3DES

Bonus points: If the HIGH list ever changes,
you most likely need to do nothing:
On most distros,
installing the OpenSSL automatic upgrade will cause all processes
using it (including your webserver) to restart,
causing it to re-evaluate its cipher list.

If you - like me - prefer EDH 256bit AES over EECDH 128 bit AES -
or worse, 112 bit 3DES -
then add `@STRENGTH` to the end of the list:
`EECDH+HIGH:EDH+HIGH:@STRENGTH`

1. EECDH 256 bits AES/Camellia
1. EDH 256 bits AES/Camellia
1. EECDH 128 bits AES/Camellia
1. EDH 128 bits AES/Cemallia
1. EECDH 112 bits 3DES
1. EDH 112 bits 3DES

This provides a sane set of cipher suites, profits from OpenSSL's
curated HIGH list
(saving you the trouble of keeping up to date what is broken)
and supports a vast majority of browsers.

Out of the ssllabs.com test, only IE6/XP, IE8/XP and Java 6u45
are *not* able to connect to a site configured like this.

* IE6/XP can only connect to SSLv2 and SSLv3,
and since both these protocols are known broken,
there is no chance we can support it.
* IE8/XP does not support forward secrecy
(at least not with >1024 bit keys),
but it supports TLSv1.0.
* Java 6u45 only supports 1024 bit DH primes

## Supporting IE8/XP
If you want to add IE8/XP, you can append
`:RSA+HIGH` to the end of the cipher list
(after `@STRENGTH`).
This ensures that all suites with forward-secrecy
get preferred, but still allows IE8/XP to connect.

IE8/XP will use a 3DES (112 bit) stream cipher.
