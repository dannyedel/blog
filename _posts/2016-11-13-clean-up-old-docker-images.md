---
title: Clean up old docker images
---

Get rid of all those docker images that creep up after a while!

```bash
docker rmi $(docker images --filter dangling=true --quiet)
```

This will remove all images that are 'dangling', i.e. not tagged
themselves or required by a tagged image as intermediates.
