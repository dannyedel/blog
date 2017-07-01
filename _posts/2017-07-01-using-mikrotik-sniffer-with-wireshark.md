---
title: Using the mikrotik built-in packet sniffer with wireshark
---

I am assuming that the mikrotik box has a direct VLAN access to the
interesting traffic.  Furthermore, the Mikrotik box **must** have
working IPv4 connectivity to the machine running wireshark.

For simplicity, I assume that the traffic towards the wireshark host
will *not* be within the VLAN being monitored.  If that is the case,
you must add filters to the sniffer so that it doesn't pick up its own
stream, creating a positive feedback loop.

First, login to the mikrotik box and execute those commands:

```mikrotik
/tool sniffer
stop
set only-headers=no streaming-enabled=yes \
  filter-stream=no streaming-server=10.11.22.33
start interface=vlan1234
```

In this example, 10.11.22.33 is the IP of the box with wireshark and
vlan1234 contains the traffic we're interested in.

On the target box, run

```bash
wireshark -i eth0 -k -f 'udp port 37008'
```

From now on, you can do the following sequence to capture on a different
interface:

```text
Mikrotik:  /tool sniffer stop
Wireshark: CTRL-R, Restart Current Capture
Mikrotik:  /tool sniffer start interface=vlan1122
```
