---
title: Using Linux's networking features to emulate a (possibly bad) WAN connection
---

When moving a LAN-connected computer from the company headquarters to an
employee's home to enable working remotely, it needs to be connected to
the company network via some kind of VPN connection.  There are lots of
possibilities how to facilitate this, including solutions that emulate
Layer-2-connectivity (for example, OpenVPN in TAP mode) which should
support any kind of network application, including those that require
broadcast connectivity.  However, a WAN connection does not just mean
lack of broadcast connectivity, it also has other
more-or-less-subtle differences to a LAN connections, in terms of speed,
latency, packet loss, packet reordering etc.
Assuming that the workplace application was not designed with WAN in
mind, it makes sense to *test* how well it will work over VPN, before
even moving the computer.
This article documents how I used a Linux-powered computer as a 'machine
in the middle' to emulate these network properties.

## Basic assumptions

ISPs regularly market their connections through download speeds, i.e.
maximum reachable bulk TCP throughput towards to subscriber.

Real-life experiments show that they also make excessive use of buffering,
which slightly benefits bulk data throughput, but severely impacts
interactivity, at least within the same stream.

Also, in the opposite direction, the subscriber-site DSL modems often
have large (sometimes in the order of megabytes) output buffers on their
own, again helping with saturating the ATM link, but impacting
interactivity -- if the buffer is full with a file upload, it can take
several seconds (!) until a new realtime packet gets transmitted.

To combat this problem,  router-modem combination devices will do the
buffering in software, splitting packets into different streams and
splitting the upstream between the streams, i.e. giving the voice stream
and the bulk transfer stream equal chances to send data.

While this will work fine with streams that can be differentiated by the
router (different source MAC, different IP protocol, different TCP/UDP
port, etc), it will *not* work as soon as VPN comes into play.
The now-remote workstation will open a VPN connection to the company
headquarters, which is opaque to the router.  From the router's point of
view, it is one constant stream of UDP data, between one constant IP/port at
the workstation and one constant IP/port at the company headquarters.

So our fake DSL connection needs to emulate the following properties:

* Speed limiting.
  * Note that downstream and upstream speeds may need to be set
    differently, depending on the headquarter's and the worker's
    internet connection.
* Packet buffering
  * When incoming packets exceed the line speed, we assume buffering
    (worse for interactivity) instead of dropping.
  * Assuming a realistic, but bad-case scenario, the buffer can be in
    the order of a megabyte.
  * Since we assume that the router will not differentiate between
    different streams, this must be a FIFO queue for maximum realistic
    effect.
* Delay
  * Since we're now sending packets over a much larger distance, we need
    to add some baseline delay.  This should be on the order of
    10-30ms, assuming reasonable connections on both sides.
  * If there is a wireless connection involved, add a random delay
  * Note that this baseline delay is orders of magnitude lower than the
    effective delay of a full modem buffer!
* Packet loss
  * While losing packets is not nearly as much a problem to most
    interactive networking applications as increased delay,
    it will occur with some baseline probability if the backbone routers
    are overloaded.
  * The base chance should be low but pesent, with a high correlation, i.e.
    an overloaded router will most likely drop several packets in a row.
  * Because of the massive buffering network-side, this normally only
    occurs during peak hours, when ISPs oversell their capacity and the
    routers start dropping packets.
* Packet duplication
  * While these exist in real WANs, most well-working network
    applications will not be affected by them, since they will simply
    ignore duplicate packets.
* Packet corruption
  * Sometimes a packet will arrive with a few bits flipped.
    This is very unlikely in real-world WANs, since packets with random
    bits flipped (and thus incorrect layer-2-checksum) will get dropped by
    the next router.
* Packet reordering
  * Because of the large buffers, this is a concern.
    However, since these only affect the TCP layer, they will feel
    identical to delay on the application layer.

## Setup

Essentially, we'll add a laptop with two network cards between the workstation
and the current LAN.  Of course, it needs to have two ethernet cards.

Using Linux, we'll configure the ethernet cards as a bridge so the
laptop will (initially) act as a switch.  Then we'll add tc commands to
emulate the connection issues, making it a network emulator switch.

### Preparing the laptop

```bash

# Turn off network-manager, since we will directly control eth0/eth1
systemctl stop network-manager

# Connect eth0 to the LAN and eth1 to the workstation

ip link set eth0 up
ip link set eth1 up
brctl addbr br0
brctl setfd br0 0
brctl addif br0 eth0 eth1
```

Now check whether the network works as expected.

For a basic check, `iperf3` and `ping -f` can be used.

## Adding the network emulation

This script will set up the fake network.

```bash
#!/bin/bash


# Speed of the emulated connection
echo "Downstream:  ${DOWNSTREAM:=6.0mbit}"
echo "Upstream:    ${UPSTREAM:=1.0mbit}"

# Maximum buffer length in seconds
echo "MaxBuffer:   ${MAXBUFFER:=10s}"

# Maximum burst (number of bytes that can go at once)
# Don't set this smaller than the network interface MTU
#
# Must be set much higher than MTU on multi-mbps links
echo "Burst:       ${BURST:=10k}"

# Maximum number of packets in netem's delay buffers
echo "Limit:       ${LIMIT:=1000}"

# Baseline latency and deviation
# Note this is the one-way latency.
# Round Trip Time will be twice this latency.
echo "Latency:     ${LATENCY:=30ms}"
echo "LatencyDev:  ${LATENCYDEV:=10ms}"
echo "LatencyCorr: ${LATENCYCORR:=25%}"

# Packetloss and correlation
echo "Loss:        ${LOSS:=3%}"
echo "LossCorr:    ${LOSSCORR:=75%}"

# Duplication and corruption
echo "Duplicate:   ${DUPLICATE:=0.1%}"
echo "Corrupt:     ${CORRUPT:=0.1%}"

echo "Uplink:      ${UPLINK:=eth0}"
echo "Downlink:    ${DOWNLINK:=eth1}"

# $UPLINK is for sending data towards 'the internet'
# $DOWNLINK is for sending data to the workstation

# Delete possible pre-existing qdiscs

set -ex

# uplink (towards internet)
tc qdisc replace dev $UPLINK root handle 1 \
  netem limit $LIMIT \
    delay $LATENCY $LATENCYDEV $LATENCYCORR \
    loss $LOSS $LOSSCORR \
    duplicate $DUPLICATE \
    corrupt $CORRUPT
tc qdisc replace dev $UPLINK parent 1: handle 2 \
  tbf \
    rate $UPSTREAM \
    burst $BURST \
    latency $MAXBUFFER

# downlink (to workstation)

tc qdisc replace dev $DOWNLINK root handle 1 \
  netem limit $LIMIT \
    delay $LATENCY $LATENCYDEV $LATENCYCORR \
    loss $LOSS $LOSSCORR \
    duplicate $DUPLICATE \
    corrupt $CORRUPT
tc qdisc replace dev $DOWNLINK parent 1: handle 2 \
  tbf \
    rate $DOWNSTREAM \
    burst $BURST \
    latency $MAXBUFFER
```

This script will emulate a quite unfriendly network.  Remember to set
downstream and upstream speed to the minimum of both sides!

If your headquarters has 12mbit/2mbit and your workstation has
6mbit/1mbit, then your effective downstream is only 2mbit!

### Watching data

This snippet will monitor the emulation.
Replace eth0 and eth1 with the correct interfaces.

```bash
#!/bin/bash

watch -n1 -d tc -s qdisc ls dev eth0 \; echo \; \
  echo \; tc -s qdisc ls dev eth1
```

### Stopping the network emulation:

```bash
#!/bin/bash

tc qdisc del dev eth0 root
tc qdisc del dev eth1 root
```

## Some example values

### Bad WLAN

Assuming the above script is named netem.sh, have fun with these.

Emulating a fast, but lossy local wireless network that does not
use retransmissions correctly:

```bash
MAXBUFFER=0.3s LOSS=10% LOSSCORR=1% \
  UPSTREAM=50mbit DOWNSTREAM=50mbit \
  LATENCY=10ms LATENCYDEV=1ms ./netem.sh
```

TCP throughput in this will be less than 1 mbit/s!

Now lets emulate retransmissions using random variations in delay

```bash
MAXBUFFER=0.3s LOSS=0.1% LOSSCORR=25% \
  UPSTREAM=50mbit DOWNSTREAM=50mbit \
  LATENCY=50ms LATENCYDEV=50ms LATENCYCORR=0% ./netem.sh
```

If you run iperf3, you will see how Linux slowly turns up the window
size until it is able to use the full bandwidth.  Note that this takes
almost 10 seconds.

```text
Connecting to host 10.46.45.233, port 5201
[  4] local 10.46.45.21 port 46128 connected to 10.46.45.233 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   243 KBytes  1.99 Mbits/sec    1   21.2 KBytes       
[  4]   1.00-2.00   sec   373 KBytes  3.06 Mbits/sec    0   31.1 KBytes       
[  4]   2.00-3.00   sec   527 KBytes  4.32 Mbits/sec    0   42.4 KBytes       
[  4]   3.00-4.00   sec   673 KBytes  5.51 Mbits/sec    0   52.3 KBytes       
[  4]   4.00-5.00   sec   788 KBytes  6.45 Mbits/sec    0   74.9 KBytes       
[  4]   5.00-6.00   sec  1.27 MBytes  10.6 Mbits/sec    0    122 KBytes       
[  4]   6.00-7.00   sec  1.86 MBytes  15.6 Mbits/sec    0    187 KBytes       
[  4]   7.00-8.00   sec  2.84 MBytes  23.8 Mbits/sec    0    276 KBytes       
[  4]   8.00-9.00   sec  4.01 MBytes  33.6 Mbits/sec    0    382 KBytes       
[  4]   9.00-10.00  sec  5.18 MBytes  43.5 Mbits/sec    0    530 KBytes       
[  4]  10.00-11.00  sec  6.00 MBytes  50.3 Mbits/sec    0    707 KBytes       
[  4]  11.00-12.00  sec  5.79 MBytes  48.6 Mbits/sec    0    916 KBytes       
[  4]  12.00-13.00  sec  5.74 MBytes  48.1 Mbits/sec    0    979 KBytes       
[  4]  13.00-14.00  sec  5.66 MBytes  47.5 Mbits/sec    0   1.08 MBytes       
[  4]  14.00-15.00  sec  5.63 MBytes  47.2 Mbits/sec    0   1.25 MBytes       
[  4]  15.00-16.00  sec  5.70 MBytes  47.9 Mbits/sec    0   1.66 MBytes       
[  4]  16.00-17.00  sec  5.77 MBytes  48.4 Mbits/sec    0   1.66 MBytes       
[  4]  17.00-18.00  sec  5.59 MBytes  46.9 Mbits/sec    0   1.66 MBytes       
[  4]  18.00-19.00  sec  5.73 MBytes  48.1 Mbits/sec    0   1.66 MBytes       
[  4]  19.00-20.00  sec  5.82 MBytes  48.8 Mbits/sec    0   1.66 MBytes       
[  4]  20.00-21.00  sec  5.60 MBytes  47.0 Mbits/sec    0   1.66 MBytes       
[  4]  21.00-22.00  sec  5.64 MBytes  47.3 Mbits/sec    0   1.66 MBytes       
[  4]  22.00-23.00  sec  5.62 MBytes  47.2 Mbits/sec    0   1.66 MBytes       
[  4]  23.00-24.00  sec  5.77 MBytes  48.4 Mbits/sec    0   1.66 MBytes       
[  4]  24.00-25.00  sec  5.78 MBytes  48.5 Mbits/sec    0   1.66 MBytes       
[  4]  25.00-26.00  sec  5.68 MBytes  47.6 Mbits/sec    0   1.66 MBytes       
[  4]  26.00-27.00  sec  5.84 MBytes  49.0 Mbits/sec    0   1.66 MBytes       
[  4]  27.00-28.00  sec  5.64 MBytes  47.3 Mbits/sec    0   1.66 MBytes       
[  4]  28.00-29.00  sec  5.66 MBytes  47.5 Mbits/sec    0   1.66 MBytes       
[  4]  29.00-30.00  sec  5.71 MBytes  47.9 Mbits/sec    0   1.66 MBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-30.00  sec   132 MBytes  36.9 Mbits/sec    1             sender
[  4]   0.00-30.00  sec   131 MBytes  36.7 Mbits/sec                  receiver

iperf Done.
```

### Bad UMTS

German mobile internet *before* limit kicks in

```bash
MAXBUFFER=30s LOSS=0.1% LOSSCORR=0% \
  DUPLICATE=0% CORRUPT=0% \
  UPSTREAM=1mbit DOWNSTREAM=1mbit \
  LATENCY=150ms LATENCYDEV=50ms LATENCYCORR=75% \
  ./netem.sh
```

German mobile internet *after* limit kicks in
```bash
MAXBUFFER=30s LOSS=0.1% LOSSCORR=0% \
  DUPLICATE=0% CORRUPT=0% \
  UPSTREAM=32kbit DOWNSTREAM=32kbit \
  LATENCY=150ms LATENCYDEV=50ms LATENCYCORR=75% \
  ./netem.sh
```

Because of the low overall packet loss, TCP bulk throughput reaches
the advertised bandwidth.   However, as soon as any bulk
transfer is active, latency goes up to 20-30 seconds, giving the
appearance of a dead connection since new connections will time out on
the TCP 3-way-handshake state or will time out on DNS resolves.

If you set the maximum delay buffer line to something more resonable
(i.e. less than one second), you get a lot more responsive network
connection, that reminds of how a 56k modem actually felt.

56k modem emulation
```bash
MAXBUFFER=0.1s LOSS=0.1% LOSSCORR=0% \
  DUPLICATE=0.1% CORRUPT=0.5% \
  UPSTREAM=56kbit DOWNSTREAM=56kbit \
  LATENCY=20ms LATENCYDEV=3ms LATENCYCORR=75% \
  BURST=1560 ./netem.sh
```

### Realistic ADSL VPN link

Common ADSL-to-ADSL VPN

```bash
MAXBUFFER=0.1s LOSS=1% UPSTREAM=1mbit DOWNSTREAM=1mbit \
  LATENCY=30ms LATENCYDEV=5ms LOSS=1% ./netem.sh
```
