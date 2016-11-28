---
title: Automatic SSH agent key handling
---

I want to automatically add my ssh key into the agent when I connect to
some external host and get prompted for my key's passphrase.  On the
other hand, I don't want it to stay in memory forever.

## user's ~/.ssh/config

Add this (before any host-specific options) to ensure all keys get added
to the agent when used.
See `man 5 ssh_config` for details.

```text
AddKeysToAgent yes
```

The keys added use the **agent's** default lifetime.  If not otherwise
configured, this default lifetime is "forever".

## ssh-agent's command line parameters

On Debian systems, your `ssh-agent` likely gets autostarted by Xsession
when you log in graphically.  Just run a `grep -HnRw ssh-agent /etc` to
find a few places where the agent might get called and then edit the
Right One™.

The command-line parameter we want to pass is `-t <lifetime>`, see
`man 1 ssh-agent` for detials.

On my system, The file is `/etc/X11/Xsession.d/90x100-common_ssh-agent`
and adding the following line did the trick.

```bash
SSHAGENTARGS="$SSHAGENTARGS -t 12h"
```
