---
---

Having a personal Debian repository is quite handy - I use it (among other
things) to push metapackages so that I can keep the stuff installed on my
systems in sync.  This is about the behind-the-scenes work that keep it going.

I want to have a server running which - similar to the debian.org hosts -
accepts my `.changes` file via anonymous upload and then decides based on
whether it was gpg signed with my key, what to do with it.

## Web server

With nginx, creating a upload-only directory works like this:

```nginx
location /incoming {
	# Allow PUT to place files on server
	dav_methods PUT;
	# Make the resulting files world-writable
	dav_access all:r;
	# Deny anything except uploading
	# (specifically, this forbids HEAD or GET requests)
	limit_except PUT {
		deny all;
	}
}
```

## Basic reprepro configuration

I created a user specifically for reprepro.  This makes gpg management quite
easy, since this users' default gpg key is just the one that signs the packages
and to add a key to the keyring (to verify uploaders' keyrings) I don't need to
juggle with `--gpghome` or other commandline magic.

1. Create a directory `/home/username/reprepro`
1. Inside that, create a `conf` directory
1. Create the following files in the `conf` directory:
   1. `distributions`
   1. `uploaders`
   1. `incoming`

What the files *do*, please read the full docs (`man reprepro`).  This is about
the minimal subset to get stuff working.

### distributions

Minimal working example:

```conf
Codename: unstable
Architectures: source amd64
Components: main
SignWith: default
Uploaders: uploaders
```

### uploaders

```conf
allow * by key ABCD0123456789ABC
```

### incoming

```conf
Name: some-name
IncomingDir: incoming/
TempDir: /some/temporary/directory
Allow: unstable
Cleanup: on_deny on_error
```

## Testing basic setup

Run, as the prepared user, both these commands:

```
reprepro -b /path/to/repository -VVVVV export
reprepro -b /path/to/repository -VVVVV processincoming some-name
```

The first command should generate a correctly signed, but empty repository.
Check this with your apt client after adding the gpg key of the packager user.

The second command will do nothing, provided you typed everything right.

## Testing uploads interactively

Upload a file with `dput-ng` to the incoming directory.  Config snippet:

```ini
[myhost]
method = https
fqdn = my.package.servers.fqdn
incoming = /path/to/incoming/folder
allow_unsigned_uploads = 1
```

Note the `allow_unsigned_uploads=1`.  We'll change that later on.  Now upload a
`.changes` which is *not* signed, and check with the processincoming command.
It should pick up the changes, but decide that it's not allowed to be included
(because it's not signed, aha!).  Try the same with an invalid key if you want.

Then sign it with your intended key, upload, and run processincoming again.  If
it all works out, great!  Onto the next step.

## Fully automated uploading

There's a nice program called `inoticoming` which will use the `inotify` system
to watch the directory.

With a little `systemd` magic, we can make it watch our `incoming` directory and
run a script (which will come down to reprepro with some parameters) as soon as
a `.changes` file comes in.

### Example script

```bash
#!/bin/bash

# /home/USERNAME/processincoming.sh

set -e
set -o pipefail
mkdir -p $HOME/logs

reprepro --basedir DIRECTORY \
	--waitforlock 1000 -VVVVV \
	processincoming some-name \
	"$@" 2>&1 | \
	tee -a $LOGFILE | \
	mailx -E -s "reprepro incoming report for $@" ADMINS@EMAIL.ADDRESS
```

Mailx's `-E` parameter means to discard the whole mail if the body is empty.
While this should™ not happen (if given a `.changes` file, it should™ to
something) I don't see the point in receiving empty emails.

### systemd unit

Place code similar as this in `/etc/systemd/system/inoticoming.service`
and execute `systemctl daemon-reload` to pick up the changes in the file.

```ini
[Unit]
Description=Automatic incoming/ handler
After=network.target

[Service]
User=USERNAME
Type=forking
ExecStart=/usr/bin/inoticoming \
	--pid-file /home/USERNAME/inotipid \
	--initialsearch \
	DIRECTORY/incoming \
	--suffix .changes \
	/home/USERNAME/processincoming.sh {} \;
PIDFile=/home/USERNAME/inotipid
Restart=always
RestartSec=15

[Install]
WantedBy=multi-user.target
```

## Final test

Now use `dput-ng` to drop a `.changes` into the folder.  You should receive an
email within seconds, giving you a reasonably verbose output of what happened to
the repository, or why your upload was rejected.
