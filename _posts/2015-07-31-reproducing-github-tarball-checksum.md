---
title: bit-exact reproduction of a github tarball for verification
---

## Rationale

Verifying that a downloaded archive of computer software is free
from corruption - or worse, intentional manipulation -
is an important part of computer security.

I release my software by creating gpg-signed `git tag`s.

For most source-based distributions,
github's tarball generator will be
the preferred method of grabbing the sourcecode.

So we need a method to verify it was created exactly from
the gpg tag I signed, here's a step-by-step instruction:

## Import my gpg key

My key should be on pretty much any keyserver by now, since
many of them are linked. However, I found the sks-keyservers pool
to be pretty reliable, so its provided as an example.

```bash
gpg --keyserver hkp://hkps.pool.sks-keyservers.net --recv-keys 0x7183343C
```

If you're not able to verify who this key actually belongs to,
at least you'll be able to detect accidental corruption and upcoming
man-in-the-middle attacks, provided they *have not started yet*.
This is similar to how SSH works, and you can read a wikipedia
article on [trust on first use] for the benefits.

tl,dr: It's better than *no verification at all*.

[trust on first use]: https://en.wikipedia.org/wiki/Trust_on_first_use

## Grabbing sourcecode and verifying the gpg tag

Grab the sourcecode with git and verify the tag's integrity.

```bash
cd $(mktemp -d)
git clone git://github.com/dannyedel/dspdfviewer .
git tag --verify v1.13
```

This will say something like

```text
 (... snip ...)
gpg: Signature made Thu 30 Jul 2015 17:07:30 CEST using RSA key ID 7183343C
gpg: Good signature from "Danny Edel <mail@danny-edel.de>"
```

Now that you know that the tag `v1.13` is indeed from me, all that is left
is reproducing the github tarball and comparing them bit-by-bit.

If you inspect the github tarball, all files are prefixed with `dspdfviewer-1.13/`,
as is customary for source tarballs.

Note that they stripped the `v` from the tag name. Very nice.

So lets compare a locally generated tarball to github's tarball **when decompressed.**

```bash
git archive v1.13 --format=tar --prefix='dspdfviewer-1.13/' -o local.tar && \
curl --silent -L https://github.com/dannyedel/dspdfviewer/archive/v1.13.tar.gz | gzip -dc > remote.tar && \
sha256sum local.tar remote.tar
```

Example output:

```text
fc776a179fc0fb3e13f172d52fb57be44dc79a70f6ffd11ff900faca1704e972  local.tar
fc776a179fc0fb3e13f172d52fb57be44dc79a70f6ffd11ff900faca1704e972  remote.tar
```

Okay, so we already know the *uncompressed* contents of the package are bit-identical.
That's good. Lets try a `.tar.gz` (compressed) file.

If you run the following command repeatedly, you will notice **a different checksum
every second**. That's because gzip embeds the filename (stdin) and the
current time into the archive:

```bash
git archive v1.13 --format=tar --prefix='dspdfviewer-1.13/' | gzip | sha256sum
```

The manpage tells us that `gzip -n` will *not* save the original file name
and timestamp by default. So let's add this:

```bash
git archive v1.13 --format=tar --prefix='dspdfviewer-1.13/' | gzip -n | sha256sum
```

Now the checksum is stable. Just for a final check, let's compare this to the github
tarball:

```bash
git archive v1.13 --format=tar --prefix='dspdfviewer-1.13/' | gzip -n > local.tar.gz && \
curl --silent -L https://github.com/dannyedel/dspdfviewer/archive/v1.13.tar.gz > remote.tar.gz &&\
sha256sum local.tar.gz remote.tar.gz
```

```text
4bf218014b88a6944dbc2e7c3cc4fa24477cbf62a077a4004c0eea500923319d  local.tar.gz
4bf218014b88a6944dbc2e7c3cc4fa24477cbf62a077a4004c0eea500923319d  remote.tar.gz
```

Success!

**You can use this way to generate a gpg-verified checksum to include in packaging recipes.**

## tl,dr: Copy and Paste

This assumes you'll download and the file `v1.13.tar.gz` to `dspdfviewer-1.13.tar.gz`, which
is most likely the default for most distributions.

```bash
# edit here. Probably works for other projects too.
PROJECT=dspdfviewer
OWNER=dannyedel
GITTAG=v1.13
GPGKEY=0x7183343C

# normally no need to edit here
VERSION=${GITTAG/#v/}
FILENAME="$PROJECT-$VERSION.tar.gz"


# no more need to edit here normally
cd $(mktemp -d) && \
gpg --keyserver hkp://hkps.pool.sks-keyservers.net --recv-keys "$GPGKEY" && \
git clone --bare "git://github.com/$OWNER/$PROJECT.git" && \
git -C "$PROJECT.git" tag --verify "$GITTAG" && \
git -C "$PROJECT.git" archive "$GITTAG" --format=tar --prefix="$PROJECT-$VERSION/" | gzip -n > "$FILENAME" && \
for chksum in md5sum sha1sum sha256sum ; do
	$chksum "$FILENAME" | tee -a DIGESTS
done
```

Example `DIGESTS` file:

```text
48dd79ae968a06eb5b58ce8d182eb4b2  dspdfviewer-1.13.tar.gz
39807c2cd0db39578fe78d6c592b61d776f1fad6  dspdfviewer-1.13.tar.gz
4bf218014b88a6944dbc2e7c3cc4fa24477cbf62a077a4004c0eea500923319d  dspdfviewer-1.13.tar.gz
```
