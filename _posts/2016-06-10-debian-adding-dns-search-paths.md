---
title: "Debian: adding some dns search paths independent of DHCP"
---

I often need to SSH into some hosts that have a FQDN similar to
`some-host-name.servers.department.example.com`.
While it makes sense to organize servers that way,
it's really annoying to type the full incantation of
`ssh root@some-host-name.servers.department.example.com`
every time you need to change something.

Writing `search servers.department.example.com` to `/etc/resolv.conf`
seems like the natural solution, however this file will likely
get overridden by the DHCP client -- and it is normally desirable to
have the DHCP-set domainname in the search path.

With Debian's `/etc/network/interfaces` <-> `resolvconf` integration,
this can be solved by adding a search path on the loopback interface:

```interfaces
#/etc/network/interfaces:

# The loopback network interface
auto lo
iface lo inet loopback
	dns-search server.department.example.com

# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
# This is an autoconfigured IPv6 interface
iface eth0 inet6 auto
```

In my experiments, the manually specified search from lo
always gets sorted **before** the DHCP-specified one.

In my case, this is desirable -- I want `ssh myservername`
to mean `ssh myservername.server.department.example.com`,
even if there is a `myservername.freewifi.box` or similar.

{% comment %}

## Alternative: Use .ssh/config

Of course, one can always add all hosts in `.ssh/config` with
a stance like

```ssh_config
Host some-host-name
	Hostname some-host-name.server.department.example.com
```

and never worry about DNS or DHCP-overridden-search-paths.


I personally prefer the above solution, since there is only
one place (BIND zonefile) where I have to manage the hostnames.

{% endcomment %}
