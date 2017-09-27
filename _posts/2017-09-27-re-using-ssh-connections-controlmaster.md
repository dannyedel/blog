---
title: Re-using SSH connections with ControlMaster
---

Connecting to an SSH host often takes a lot of time,
most of which is spent on authentication and key exchange.
This effect is especially noticable
on high-latency connections or when connecting to
low-cpu servers or embedded systems.

Using the "ControlMaster" mode you can re-use the connection,
resulting in a lot less overhead for the next connections.

Also, as a side-effect, when using password-authentication you
only have to type the password once, as long as the master connection
is alive.

`~/.ssh/config`:

```text
Host *
	ControlMaster auto
	ControlPath ~/.ssh/cm_%C
	ControlPersist 10m
```

The `%C` parameter contains the local hostname's FQDN, remote hostname,
remote port and the remote username.  At least on my system, it simply
outputs a hash value.

Make sure the path chosen is not readable by another user.

-----

## How effective is it?

As an example of the low-cpu category, I'll run the "date" command
twice on a low-end TP-Link running on OpenWrt -- first with
ControlMaster disabled, and then with it enabled.

The authentication is done using an RSA key.

```bash
/usr/bin/time bash -c '\
	ssh -o ControlMaster=no root@$TPLINK date && \
	ssh -o ControlMaster=no root@$TPLINK date'
```

Output:

```text
Wed Sep 27 13:17:34 CEST 2017
Wed Sep 27 13:17:38 CEST 2017
0.06user 0.00system 0:08.26elapsed 0%CPU (0avgtext+0avgdata 6128maxresident)k
0inputs+0outputs (0major+780minor)pagefaults 0swaps
```

Same commands, but with `ControlMaster=auto`:

```text
Wed Sep 27 13:18:47 CEST 2017
Wed Sep 27 13:18:48 CEST 2017
0.03user 0.00system 0:03.53elapsed 1%CPU (0avgtext+0avgdata 6184maxresident)k
0inputs+0outputs (0major+807minor)pagefaults 0swaps
```

Note that the time got cut about in half (8sec down to 4sec),
since most of the time was spent establishing the link.

For reference, this is how long the command takes with
`ControlMaster=auto` and the connection already established.

```text
Wed Sep 27 13:20:32 CEST 2017
Wed Sep 27 13:20:32 CEST 2017
0.01user 0.00system 0:01.52elapsed 0%CPU (0avgtext+0avgdata 5812maxresident)k
0inputs+0outputs (0major+745minor)pagefaults 0swaps
```
