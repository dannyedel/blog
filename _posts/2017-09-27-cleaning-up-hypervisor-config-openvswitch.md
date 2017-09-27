---
title: Cleaning up hypervisor's network config with Open vSwitch
---

My homeserver acts as a hypervisor for some VMs, which connect to
different VLANs.

Previously, I had a hand-crafted configuration on my (debian-based)
hypervisor, which looked essentially like this:

```text
auto vlan123
iface vlan123 inet manual
	vlan_raw_device eth0

auto bridge123
iface bridge123 inet manual
	bridge-ports vlan123
	bridge-fd 0


auto vlan124
iface vlan124 inet manual
	vlan_raw_device eth0

auto bridge124
iface bridge124 inet manual
	bridge-ports vlan124
	bridge-fd 0


auto vlan125...
```

And so on for every VLAN my VMs were allowed to connect to.

On the bridge of the management VLAN, I would assign the server its own
IP address.

This also resulted in the side-effect of the hypervisor listening to
IPv6 traffic on the VM instances, and with the default configuration
that meant it would accept a router advertisement from a VM, which can
result in connectivity loss.

## Change to Open vSwitch as bridge backend

Open vSwitch (`ovs`) is essentially a front-end to controlling switch
logic, and it can also use the Linux kernel's software bridge as a
back-end.

Using this instead of manually configuring VLANs and bridges on them
has a few advantages:

* The Hypervisor's `/etc/network/interfaces` file is no longer cluttered
  with VM-related configuration
* Open vSwitch will respect VLAN boundaries, **hiding** the VM traffic
  in other VLANs from the hypervisor (unless explicitly asked to mirror
  a port)
* Since the VMs' VLANs are not listed in the host's `interfaces` file,
  adding or removing VLANs does not affect the host

(Note: As usual, I'm using a Debian host.  Since network configuration
is highly distribution-specific, this section will likely not work on
other operating systems.)

### Install packages and stop VMs

Install packages `openvswitch-common` (management utils) and
`openvswitch-switch` (backend).

```bash
apt install openvswitch-common openvswitch-switch
```

Also, you should **stop** (not suspend) any VMs, since we'll be changing
their network setup, and as far as I know you can't resume a VM if the
bridge it is supposed to connect to is gone.

### Configure /etc/network/interfaces

Open vSwitch is integrated into Debian's network interfaces file well
enough to use that entirely for configuration.

Important:  Drop the word `auto` and use `allow-ovs` instead.  At least
on Debian jessie, the network interfaces will *not* come up when you use
`auto`.  This can be extra annoying if you didn't setup an out-of-band
management backdoor.

As an additional note, I will **disable IPv6** in general and allow it
only on the management interface.  This avoids cluttering `ip -6 route`
with all the `vnet` interfaces from the VMs.

```text
auto lo
iface lo inet loopback

allow-ovs ovsbr0
iface ovsbr0 inet manual
        ovs-type OVSBridge
        ovs-ports eth0 vlan1234

allow-ovsbr0 eth0
iface eth0 inet manual
        ovs-bridge ovsbr0
        ovs-type OVSPort

allow-ovsbr0 vlan1234
iface vlan1234 inet static
        ovs-bridge ovsbr0
        ovs-type OVSIntPort
        ovs-options tag=1234
        address 10.2.3.4
        netmask 255.255.255.0
        gateway 10.2.3.1
        dns-nameservers 10.2.3.1

allow-ovsbr0 vlan1234
iface vlan1234 inet6 auto
        privext 2
        accept_ra 1
        pre-up sysctl net.ipv6.conf.vlan1234.disable_ipv6=0
```

Word of warning:  ***CASE IS IMPORTANT*** It is `OVSBridge`, **not**
`ovsbridge` or `OvsBridge`.

This sets up a `OVSBridge` named `ovsbr0` consisting of two ports:
* eth0, the `OVSPort` (uplink to the outside world)
* vlan1234, an `OVSIntPort` (internal port, used by the hypervisor)
  The `ovs-options tag=1234` sets the VLAN-ID on this.
   * Static IPv4
   * Dynamic (SLAAC) IPv6, with privacy extensions enabled

This concludes the setup of the hypervisor.


After rebooting, you can check the ovs state with `ovs-vsctl show`.

Example output:

```text
    Bridge "ovsbr0"
        Port "vlan205"
            tag: 205
            Interface "vlan205"
                type: internal
        Port "eth0"
            Interface "eth0"
        Port "ovsbr0"
            Interface "ovsbr0"
                type: internal
    ovs_version: "2.3.0"
```

### Setup LibVirt VMs

Connecting a VM to a VLAN with Open vSwitch is very similar to
connecting it to a regular bridge.  Unfortunately, my favorite GUI
`virt-manager` does not yet (version 1.4.0) support setting the VLAN
IDs.  However, once they're set by editing the XML,
the GUI does not seem to delete them.

So **once per VM** the XML needs to be edited.

You need to

1. Tell libvirt the name of the new bridge
2. Tell it that this is not a regular, but an ovs bridge
3. Configure VLANs for the VM

Note that the packets will always leave tagged on eth0, since this
is the default mode for an OVSPort.

```xml
<!-- ... -->
<interface type='bridge'>
	<!-- Update this entry, set correct source bridge name -->
	<source bridge='ovsbr0' />
	<!-- Set the bridge control mode to ovs -->
	<virtualport type='openvswitch' />

	<!-- Example for an access VLAN:
	  * From the VMs perspective, this interface is untagged
	-->
	<vlan>
		<tag id='1011' />
	</vlan>

	<!-- Example for a trunk VLAN:
	  * VM sees tagged packets
	-->
	<vlan trunk='yes'>
		<tag id='1011' />
		<tag id='1012' />
	</vlan>

	<!-- Example for a hybrid access/trunk VLAN:
	  * VM sees VLAN 101 as untagged packets
	  * VM sees VLAN 1011 and 1012 as tagged packets
	-->
	<vlan trunk='yes'>
		<tag id='101' nativeMode='untagged' />
		<tag id='1011' />
		<tag id='1012' />
	</vlan>

	<!-- model, PCI address, MAC, etc... -->
</interface>
```

Note that the hybrid configuration is great for bootstrapping
virtual network applicances that spawn some kind of configuration
interface and possibly a DHCP server on first boot.

When first setting up, they will spawn in the 'native' VLAN,
so you can hook up your laptop with a DHCP client to that VLAN.

Then you configure them to use VLAN tagged packets and integrate
them into the proper network setup.

Note that I did *not* have to tell Open vSwitch in advance about
the VLANs I will be using with libvirt.

The VLAN configuration is now purely in the virtual machine's XML file
and can migrate to a second, similarly-configured hypervisor without
changing any configuration on either host.
