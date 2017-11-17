---
title: Using the DisplayLink USB-to-DVI with Nvidia
---

Initially for the (Windows-running) work machine,
I ordered an USB-to-DVI-Adapter which was described by the dealer
as "This does not work with Linux!!!".

Of course, if it's listed with three exclamation marks,
it must be totally true.

So I plugged it in anyway, and checked what happens...

## What device is it, exactly?

It was marketed as "I-TEC USB 2.0 Display Video Adapter DVI HDMI VGA
FullHD 1920x1080p Externe Monitor Grafikkarte".

`lsusb -vv` says that it's a DisplayLink DL-165 Adapter:

```text
Bus 002 Device 002: ID 17e9:0290 DisplayLink 
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            0 (Defined at Interface level)
  bDeviceSubClass         0 
  bDeviceProtocol         0 
  bMaxPacketSize0        64
  idVendor           0x17e9 DisplayLink
  idProduct          0x0290 
  bcdDevice            0.01
  iManufacturer           1 DisplayLink
  iProduct                2 DL-165 Adapter
  iSerial                 3 826156
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength           73
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0 
    bmAttributes         0xc0
      Self Powered
    MaxPower              500mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           3
      bInterfaceClass       255 Vendor Specific Class
      bInterfaceSubClass      0 
      bInterfaceProtocol      0 
      iInterface              0 
      ** UNRECOGNIZED:  22 5f 01 00 20 05 00 01 03 04 02 04 ff 59 62 02 00 02 04 00 bd 1f 00 01 04 01 02 00 04 04 01 00 03 d0
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x01  EP 1 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0200  1x 512 bytes
        bInterval               0
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x82  EP 2 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0008  1x 8 bytes
        bInterval               4
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x0a  EP 10 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0200  1x 512 bytes
        bInterval               0
Device Qualifier (for other device speed):
  bLength                10
  bDescriptorType         6
  bcdUSB               2.00
  bDeviceClass            0 (Defined at Interface level)
  bDeviceSubClass         0 
  bDeviceProtocol         0 
  bMaxPacketSize0        64
  bNumConfigurations      1
Device Status:     0x0000
  (Bus Powered)
```

## What does not work

Now **at least with the NVIDIA binary driver**,
you cannot use an additional **GPU** unless it's in supported
by the exact same nvidia driver module that you currently use.

It's possible that this restriction does not apply with open source
KMS drivers, but I have an nvidia card and use the closed-source drivers
to have hardware-accelerated video decoding and deinterlacing.

So we can't use the new GPU to add an independent frame buffer with its
own drawing capabilities.  Note that this is true regardless of the
connector, for example you can't plug in your old PCI Voodoo card and
use it in addition to the Nvidia.

## What DOES work

The X Server nowadays includes support for image offloading.
Initially, this was used for laptops with one low-energy GPU
connected to the monitor, and one high-power "gaming" GPU
that can be activated on-demand.  The gaming GPU usually has
**no direct monitor connection** and sends the computed images
to the low-energy GPU.

Using this technology, you can use the existing GPU to compute
the image into its own memory,
and then send the rendered output to the USB adapter.

This method, however good it sounds at first, has its downsides.

Since the chip's own processing is not used to generate the image,
you essentially send rendered RGB images through the USB bus.

At 1280x1024x24bit, that's about 31mbit **per frame**.
It'll get worse for larger resolutions of course --
a 1920x1200x24bit image is 55mbit per frame.

So at USB2's 480 MBit, I'd expect something around
15fps for my 1280x1024x24 images under ideal conditions,
meaning
* nothing else uses the (shared!) USB bus
* there is no USB protocol overhead whatsoever (har, har...)

The frames per second naturally could go up if the connection between
the main GPU and the USB device uses something more sophisticated than
full-frame-transfer (such as some compression scheme),
but as soon as complicated (i.e. uncompressible) material
will be rendered, the framerate is likely to approach this
low value.

Note that Windows may perform a bit better in this regard, since
(at least to my knowledge) it actually uses the USB chip's processor
to compute the image, at least to some extent.

## Kernel and NVIDIA driver versions

This offloading is fairly new stuff.  You'll want the most recent kernel and
nvidia driver you can get your hands on.  With the versions in stretch,
I had terrible cursor flickering, prompting me to upgrade the kernel to
stretch-backports.  Howver, the old nvidia-driver does not work with the
newer kernel, so I had to upgrade that too.  (There's no
nvidia-legacy-340xx in stretch-backports, so I used the buster version
directly.  So far, it's not b0rken.)

I updated my kernel to stretch-backports and used
the NVIDIA binary driver from buster.  These are the
apt pins I used:

```text
$ cat /etc/apt/preferences.d/kernel-stretch-backports
Package: linux-image-amd64 linux-headers-amd64
Pin: release n=stretch-backports
Pin-Priority: 990

$ cat /etc/apt/preferences.d/nvidia-buster
Package: *nvidia-legacy-340xx*
Pin: release n=buster
Pin-Priority: 990
```

This results in Linux version `4.13.4-2~bpo9+1` and nvidia
version `340.104`.

## Prevent udlfb from loading

Thanks to the [Arch Linux wiki][1], I knew about the
old driver blocking the USB interface and disabled it **before**
I plugged it in and it got auto-loaded.

```bash
###   Do this before plugging in, or you have to reboot.

# Blacklist udlfb driver, only the udl driver should load
echo 'blacklist udlfb' > /etc/modprobe.d/blacklist-udlfb.conf
```

Now plug it in, and... nothing happens.  This is good.

If you forgot the above step, your new screen will turn green,
you'll try to unplug it and `rmmod udlfb`, see that it's still "in use",
and you'll have to reboot after blacklisting udlfb...

## Configure Xorg

First, verify that the new device was detected:

```bash
xrandr --listproviders
```

example output:

```text
$ xrandr --listproviders
Providers: number : 2
Provider 0: id: 0x249 cap: 0x1, Source Output crtcs: 2 outputs: 4 associated providers: 1 name:NVIDIA-0
Provider 1: id: 0x29d cap: 0x2, Sink Output crtcs: 1 outputs: 1 associated providers: 1 name:modesetting
```

This is good.  Now just define the USB device as a slave of the nvidia
and stuff will work.  Mostly.

```bash
xrandr --setprovideroutputsource modesetting NVIDIA-0
```

Now the new device is available in xrandr as a new monitor with some
more-or-less-strange name.  In my case, it was `DVI-I-1-1`.  If you
don't have a good terminal font that clearly differentiates between
`1 L I l`, now's the time to get one.

Simply activating the monitor resulted in a working image straight away.

```bash
# Activate the new monitor!
NAME=DVI-I-1-1
xrandr --output $NAME --auto --right-of HDMI-0
```

## Fix the scrolling

However, something™ (might be the nvidia driver) automatically activates
scrolling on the primary two monitors as soon as the mouse cursor enters
the no-longer-on-the-physical-connectors output area.

Trying to disable panning just results in something™ putting it back on.

Thanks to [this comment on the ubuntu bug][2], I learned that you can
set values to the `--panning` argument of xrandr which will result in
panning being active, but **never getting triggered** since the mouse
cannot reach the trigger area.

The linked post uses the 'borders' part of the `--panning`
parameter to achive that.

I found it easier to use the second part ("track") to make the first
monitor scroll to everything ("panning", first part), but only as long
as the mouse cursor is in the - you guessed it - first monitor.

In my example, my total screen width is `1920+1920+1280=5120` and my
total height is 1200, since my monitors are arranged horizontally.

```bash
# Fix scrolling.  By default, nvidia driver will make the screens
# scroll when visiting the USB screen.
LEFTMONITOR=DVI-I-1
# Total width
TOTALW=5120
TOTALH=1200
# Use left monitors width.
WIDTH=1920
HEIGHT=1200
xrandr --output $LEFTMONITOR --panning \
 ${TOTALW}x${TOTALH}+0+0/${WIDTH}x${HEIGHT}+0+0
```

## Issues left: OpenGL

Even with the new kernel and nvidia driver, there is an issue with the
mouse cursor **when hovering OpenGL windows on the primary monitor**.

The 32x32 pixel part of the monitor where the mouse cursor is **simply
stops updating underneath it**, freezing the contents it had when the
mouse pointer moved there.

Since I don't normally play openarena on my work machine, this is not
really an issue.
Especially since I'd switch to single-monitor full screen mode before
playing, which does not have any issues.

The only real-life-affecting issue is that
**Chromium seems to use OpenGL to draw when hardware-acceleration is enabled**.
So I had to turn off hardware accelerated drawing in Chromium, which
is even possible using the GUI settings.

## How good does it actually work?

For regular office work, I don't even notice the USB link and I just
like having even more space to run terminals on.
However, I wouldn't recommend playing openarena over USB2.0.

While the application says that I have 60fps, the real framerate feels
much closer to the expected (see above) 15fps.

Apparently, the frame skipping is hidden from the application, but
otherwise well implemented.

## Thanks go out to

I wouldn't have figured out where to start without
the [ArchLinux wiki][1] and [This ubuntu report][2].

[1]: https://wiki.archlinux.org/index.php/DisplayLink
[2]: https://bugs.launchpad.net/ubuntu/+source/xorg-server/+bug/1326688/comments/5
