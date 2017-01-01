---
---

For personal reference,
I want to list the screen sizes I find comfortable and those
that are too small, in order to easier judge if a proposed
device will be acceptable.

My workload mostly consists of (source) text, so having a screen
that is able to render images at or above my eyes' resolution
is pointless for me.

In my experience, too many programs do not yet work
well on high-dpi screens, generally resulting in really tiny
letters that are a pain to work with,
or they scale rendered bitmaps,
resulting in blurred on-screen images.
So generally I prefer low-dpi screens.

I will update this post with data I gather from screens
empirically, to figure out where the threshold beween "okay"
and "too small for my taste" lies exactly.

## too small to read on

* A friend's laptop
  * 16:9, 1920x1080, marketed as 15.6" diagonal
  * 1920px @ 345mm == 141dpi

## tolerable on short (laptop) viewing distance

* My current laptop
  * 16:10, 1440x900, marketed as 17"
  * 1440px @ 367mm == 100dpi

## tolerable on normal (desktop) viewing distance

* My backup secondary screen
  * 5:4, 1280x1024, marketed as 17"
  * 1280px @ 337mm == 96dpi
* My trusty SyncMaster 245B+
  * 16:10, 1920x1200, marketed as 24" diagonal
  * 1920px @ 518mm horiz. == 94dpi

## Good

* My secondary screen
  * 5:4, 1280x1024, marketed as 19" diagonal
  * 1280px @ 377mm == 86dpi
* My old laptop (asus A3HF)
  * 4:3, 1024x768, marketed as 15" diagonal
  * 1024px @ 304mm horiz. == 75dpi

## DPI Calculation

The DPI listed above is calculated based on the width measurement,
and the horizontal resolution of the screen.
This assumes square pixels, therefore ignores the height and
aspect ratio, but one can trivially measure the vertical DPI and
see if there is any difference.

```
dpi [px/inch] = imageWidth [px] / width [mm] * 25.4 [mm/inch]
```

### Guessing DPI

For a screen without actual measurements, such as one viewed
in a catalog, one has to guess the DPI if it is not given.
Formula for approximate DPI, based on marketed diagonal size,
and aspect ratio, again assuming square pixels:

```
                             imageWidth [px]
                  -------------------------------------
dpi [px/inch] =       (       diag[inch]^2        )
                  sqrt(  -----------------------  )
                      (  1 + (smallToBigRatio^2)  )

```

or, in one line,

```
dpi [px/inch] = imageWidth [px] / sqrt( ( diag [inch] )^2 * 1/(1+(smallToBigRatio)^2) )
```

Replace `smallToBigRatio` with `9/16`, `4/5`, `3/4` or `10/16` as needed.
