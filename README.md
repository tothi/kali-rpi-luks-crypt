# Full disk encryption for Kali on Raspberry using LUKS

This HOWTO is dedicated to running [Kali](https://www.kali.org/)
on [Raspberry](https://www.raspberrypi.org/) using full disk
encryption with [LUKS](https://guardianproject.info/code/luks/).
Based on the little bit outdated document
[here](https://www.offensive-security.com/kali-linux/raspberry-pi-luks-disk-encryption/).

## Motivation

Deploying Kali on Raspberry Pi in a LAN environment as a "throw-away-hackbox"
is obviously useful. However, keeping the collected data in secret is an
essential requirement. Here comes full disk encryption as a mandatory
security concept.

## Prerequisites

First we should
[setup](http://docs.kali.org/kali-on-arm/install-kali-linux-arm-raspberry-pi)
an official current Kali image for the Raspberry Pi (without encryption)
on an SD card.

The recommended option (now) is using a Raspberry Pi 3 custom build
with the [nexmon patch](https://github.com/seemoo-lab/nexmon)
(for using the onboard WiFi monitoring and injecting capabilities).
There is an automated build script in the official
[kali-arm-build-scripts](https://github.com/offensive-security/kali-arm-build-scripts)
repo which sets up the image from scratch (on a Kali distro).
And here is a
[patched version](https://github.com/tothi/kali-arm-build-scripts)
which works on other distros than Kali, too (tested on
[Gentoo](https://gentoo.org/)).

Get the build tools and build the image from scratch
as root (it can take couple of hours):
```
$ git clone https://github.com/tothi/kali-arm-build-scripts
$ cd kali-arm-build-scripts
$ sudo ./rpi3-nexmon.sh 2.0
```

The resulting image is `rpi3-nexmon-2.0/kali-2.0-rpi3-nexmon.img.xz`.
Copy it on SD card (using pv for displaying progress):
```
# pixz -d rpi3-nexmon-2.0/kali-2.0-rpi3-nexmon.img.xz - | pv -treb | dd of=/dev/mmcblk0 bs=512k
```

(The image should work, it can be tested now.)

