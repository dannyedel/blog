---
title: Verify if an ISO file was burned correctly
---

After burning, eject and re-insert the CD, and run

```bash
cmp --verbose /dev/cdrom /path/to/image.iso
```

If this outputs nothing and exits with zero, all is well.

If this outputs *EOF on /path/to/image.iso*, this is normal because the
actual CD has possibly been padded with zeros, up to the next sector
size.

Any other output most likely indicates a burning error.
