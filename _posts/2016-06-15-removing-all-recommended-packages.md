---
---

By default, when you install package A, Debian also installs packages
*recommended* by A's maintainer.
This is desirable and correct in most cases.
However, sometimes one wants to run a minimal system,
which includes only the packages *specifically* requested
or their *hard* dependencies.

To remove all others, add a file in `/etc/apt/apt.conf.d/`
containing this:

```text
APT::Install-Recommends "false";
APT::AutoRemove::RecommendsImportant "false";
APT::AutoRemove::SuggestsImportant "false";
```

And then call `apt autoremove`.

And then start requesting all those packages that you now
realize you're missing *explicitly*, either to `apt install`
or via a metapackage.
