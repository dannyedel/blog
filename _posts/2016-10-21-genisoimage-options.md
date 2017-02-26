---
title: genisoimage options
---

Some useful genisoimage options to produce a **data**
CD/DVD-image with good compatibility and not-too-insane
changes (i.e. non-clobbered path names when read on Unices)

* `-iso-level 3` (or 4, if you're adventurous)
* `-J` (Joliet) if the disk should be readable on Windows machines
* `-r` (Rock Ridge) if the disk should be reable on Unices
* `-V VolumeLabel`
* `-v` verbose

Together:

```
growisofs -Z /dev/sr0 -iso-level 3 -J -r -V DiscNameGoesHere -vv \
	/path/to/folder/containing/data/to/burn
```

For Video DVD on-the-fly burning:

```
growisofs -dvd-compat -Z /dev/sr0 -dvd-video -V VolumeName \
	/path/to/folder/containing/dvd/structure
```
