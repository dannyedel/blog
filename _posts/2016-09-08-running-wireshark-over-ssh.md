---
title: Running a wireshark capture over SSH
---

I regularly use [`wireshark`][1] if I want to take a look at network
traffic. But sometimes I need to monitor another host, probably a
rootserver somewhere.  The natural reaction is to use `tcpdump`
directly, but that often results in a very spammy console output and
makes it a bit difficult to judge the result.

The solution is to pipe the tcpdump capture result back through SSH
and live-viewing it on the workstation with wireshark.

Basic syntax:

```bash
ssh root@$HOSTNAME tcpdump -U -n -w - $MORE_OPTIONS | wireshark -k -i -
```

Explanation of the magic numbers:

* `tcpdump`
  * `-U` makes the output packet-buffered, i.e. requests SSH to send the
    output per packet, rather than on buffer full
  * `-n` supresses DNS resolves on the remote host (which might generate
    additional packets to be captured, and introduce additional delays)
  * `-w -` writes the raw data to standard output (i.e. the SSH pipe),
    instead of the pretty-printed output we normally see from tcpdump.
* `wireshark`
  * `-k` starts the capture immediately, rather than going to the splash
    screen.
  * `-i -` reads from standard input, rather than from a live interface.

***WARNING***

Make sure to run the SSH connection over a different interface than the
one you're capturing from, or specify a suitable filter in `$MORE_OPTIONS`,
otherwise you have a positive feedback loop where the packet data being
sent over SSH is itself new packet data added to the output buffer.

If your server has only one interface, a filter string like `port not
22` is mandatory.

[1]: https://tracker.debian.org/pkg/wireshark
