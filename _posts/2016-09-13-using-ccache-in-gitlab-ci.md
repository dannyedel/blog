---
title: Using CCache in GitLab-CI
---

GitLab CI is nice, and it's runner is well integrated in the rest of the
workflow.  However, unlike Travis-CI, it does not directly support
ccache, which will by default place its cache files outside the
repository.

By placing them inside the build directory, they can be picked up and
are able to speed up a build many orders of magnitude, since typically
on CI builds not much changes between commits.


```yaml
build:
    image: some-image/that-contains-ccache
    ccache:
        paths:
            - build/.ccache
    script:
        - mkdir -p build
        - cd build
        - export CCACHE_DIR=$(pwd)/.ccache
        - PATH=/usr/lib/ccache:$PATH
          cmake .. -YOUR_OTHER_CMAKE_OPTIONS_GO_HERE
        - make all
        - make test
        - ccache -s
```

Of course, the final ccache -s is not really needed, I just use it to
see how useful the cache was on the build.
