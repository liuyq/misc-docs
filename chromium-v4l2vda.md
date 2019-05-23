# Notes on Chromium V4L2 VDA enablement on Linux

These notes are taken from Linaro Connect Bangkok demo where
we demonstrated V4L2 VDA on Dragonboard db410c and 820c using
Venus v4l2 driver.

### Getting / building source code

I used the Qcom/thud rpb manifest as a starting point for this
work. Follow the instructions here
https://github.com/96boards/oe-rpb-manifest/tree/qcom/thud



### meta-browser changes

At the time of writing meta-browser is on v74 of Chromium.

I build with the following changes applied

enable prop codecs and v4l2
https://github.com/petegriffin/meta-browser/commit/3e2727dbc546d8ff62a6df1112b124118a7d4650

Build with clang toolchain (gcc is always breaking for Chromium)
https://github.com/petegriffin/meta-browser/commit/8748a31861404f5b5850201d2aea934bf71a2689

Disable v4lplugin
https://github.com/petegriffin/meta-browser/commit/f6fc7a7e60232f1a50042a82f2843ba42a66bedf

If using v4lplugin on my system at least I needed to create an extra symlink
to get things working on the rootfs.

```bash
su; ln -s libv4l2.so.0  libv4l2.so; exit
```

### Running Chromium

Chromium looks for video-dec0 device node. Also the probe order
of decoder/encoder can change (meaning video0 switches between encoder/decoder
between boots). Check this first with v4l2-compliance.

```bash
su
v4l2-compliance -d /dev/video0 | grep Card
v4l2-compliance -d /dev/video1 | grep Card

ln -s /dev/video0 /dev/video-dec0
ln -s /dev/video1 /dev/video-enc1
```

With the latest v74 Chromium many of the previous command line args
should no longer be required, but append --ignore-gpu-blacklist
so that v4l2 vda is enabled.

```bash
chromium --ignore-gpu-blacklist http://google.co.uk
```

Double check that it is using v4l2 VDA by browsing to
about://gpu page. This will list video acceleration at the top.
However also check the bottom of the page for the various codecs
it has found and enumerate from the video device.

Next is to try and play a video clip. To verify HW acceleration is
being used, open a new tab and browser to about://media-internals.
This will show the player in the other tab. Verify it says
GpuVideoDecoder and NOT FfmpegVideoDecoder.

Ffmpegvideodecoder means sw decode is being used. It is also easy
to check to see if the Venus irqs are ticking using

```bash
cat /proc/interupts | grep venus
```

### Random debugging tips

The following command line option cab be used to get extra debugging
from the gpu/media parts of chromium

```
--vmodule=*video_decode*=8,*/media/filters/gpu_video_decode*=8,*/media/*video*=8,*media/gpu/*=8,*/media/filters/gpu_video_decoder*=8*/v4l2/*=8
```

