---
title: "Dealing with NAT: Access Websites behind NAT with SSH and Chromium"
---

SSH's `-D` switch spawns a local socks proxy server which can be used
by a web browser to transparently tunnel connections.

When circumventing a local firewall issue you normally want to tunnel
all connections through the SSH proxy.

However, in this case, If I do not want to redirect *all*
my browser traffic through this SSH tunnel, but only the traffic to
certain hosts behind the firewall.

The simple solution is to spawn a second chromium profile with
the socks proxy set, and use this window only for the hosts that
need it.

```bash
# -N No remote shell
# -f Go to background after connecting
# -D 8888 Activate socks proxy server on localhost:8888
ssh -Nf -D 8888 me@my-shell-box.example.com
# Launch a chromium which uses this proxy
chromium --temp-profile --incognito --proxy-server=socks://localhost:8888 
# Kill the socks proxy when done
ssh -O cancel -D 8888 me@my-shell-box.example.com
```

Of course, `my-shell-box.example.com` could itself be behind the
firewall and might require a [ProxyJump][1] to reach.

In a sense, this is the web equivalent of SSH jump hosts.

Note that the third command will only work if a [control connection][2]
was used to establish the port forwarding.

Using temp-profile together with incognito may seem like overkill, but
it has some real-world merit:
* Chromium does not ask for google accounts
* It does not ask whether to save passwords
* It does not load my bookmarks, giving me a hint which window I'm in

-----

Bonus Points:  Figure out a way to teach chromium (or any other browser
that works well enough to administer webinterfaces of embedded devices)
to spawn the ssh process itself when needed.

[1]: /2017/10/04/dealing-with-nat-ssh-jump-host/
[2]: /2017/09/27/re-using-ssh-connections-controlmaster/
