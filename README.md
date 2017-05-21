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

## Preparing the RPi image for encrypted boot

We mount the image from the SD card and prepare an initramfs
capable with LUKS features in a chroot environment. For making
this work, an x86 (host) `qemu-arm-static` binary is needed
on the SD card filesystem. The above patched `rpi3-nexmon.sh`
build script installs and leaves it on the image, so things
should work well.

Initializing the chroot environment (on the host system as root):
```
# mkdir -p /mnt/chroot/boot
# mount /dev/mmcblk0p2 /mnt/chroot/
# mount /dev/mmcblk0p1 /mnt/chroot/boot/

# mount -t proc none /mnt/chroot/proc
# mount -t sysfs none /mnt/chroot/sys
# mount -o bind /dev /mnt/chroot/dev
# mount -o bind /dev/pts /mnt/chroot/dev/pts

# LANG=C chroot /mnt/chroot/
```

Install required packages (in the chroot env):
```
# apt-get update
# apt-get install busybox cryptsetup dropbear
```

Note, that we have to enter the decryption key on
every boot. We do it preferably remotely through SSH,
that's why `dropbear` is needed (in the initramfs).

Now edit `/boot/cmdline.txt`, change root device to
the mapped crypt device (`/dev/mapper/crypt_sdcard`)
and add the appropriate `cryptdevice` parameter.

So here is an original `/boot/cmdline.txt`:
```
dwc_otg.fiq_fix_enable=2 console=ttyAMA0,115200 kgdboc=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait rootflags=noload net.ifnames=0
```

And here is the updated one (differences in `root` and `cryptdevice` params):
```
dwc_otg.fiq_fix_enable=2 console=ttyAMA0,115200 kgdboc=ttyAMA0,115200 console=tty1 root=/dev/mapper/crypt_sdcard cryptdevice=/dev/mmcblk0p2:crypt_sdcard rootfstype=ext4 rootwait rootflags=noload net.ifnames=0
```

Now create `/boot/config.txt` with the following content:
```
initramfs initramfs.gz 0x00f00000
```

Let's set up Dropbear SSH key authentication.
Create a (passwordless) keypair on the host machine (outside chroot!)
and read the public key:
```
$ ssh-keygen -N "" -f kali-dropbear
$ cat ./kali-dropbear.pub
```

Add the public key to `/etc/dropbear-initramfs/authorized_keys`.
Restrict Dropbear SSH access for setting up cryptroot only
by prepending the key with this:
```
command="/scripts/local-top/cryptroot && kill -9 `ps | grep -m 1 'cryptroot' | cut -d ' ' -f 3`"
```

Fix permissions:
```
chmod 600 /etc/dropbear-initramfs/authorized-keys
```

Edit `/etc/fstab`. Original:
```
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
proc            /proc           proc    defaults          0       0
/dev/mmcblk0p1  /boot           vfat    defaults          0       2
/dev/mmcblk0p2  /               ext4    defaults,noatime  0       1
```

Updated `/` device changed to `/dev/mapper/crypt_sdcard`:
```
# <file system>           <mount point>   <type>  <options>       <dump>  <pass>
proc                      /proc           proc    defaults          0       0
/dev/mmcblk0p1            /boot           vfat    defaults          0       2
/dev/mapper/crypt/sdcard  /               ext4    defaults,noatime  0       1
```

Add this line to `/etc/crypttab`:
```
crypt_sdcard /dev/mmcblk0p2 none luks
```

Finally, make initramfs and leave chroot (don't bother with
errors/warnings during mkinitramfs):
```
# ls -l /lib/modules/ |awk -F" " '{print $9}'

4.4.50-v7+
# mkinitramfs -o /boot/initramfs.gz 4.4.50-v7+
# exit
```

Unmount filesystems and backup rootfs before creating encrypted
volume (and deleting everything) on SD card:
```
# umount /mnt/chroot/boot
# umount /mnt/chroot/sys
# umount /mnt/chroot/proc
# mkdir -p /mnt/backup
# rsync -avh /mnt/chroot/* /mnt/backup/
```

Unmount the full chroot:
```
# umount /mnt/chroot/dev/pts
# umount /mnt/chroot/dev
# umount /mnt/chroot
```

Delete unencrypted root partition (num. 2) and create an empty one
(fulfilling the SD card):
```
# echo -e "d\n2\nw" | fdisk /dev/mmcblk0
# echo -e "n\np\n2\n\n\nw" | fdisk /dev/mmcblk0
```

Now (probably) unplugging and inserting the SD card is needed
in order to register the new partitions.

Create encrypted volume with a strong passphrase
(this step destroys the original rootfs!):
```
# cryptsetup -v -y --cipher aes-cbc-essiv:sha256 --key-size 256 luksFormat /dev/mmcblk0p2
# cryptsetup -v luksOpen /dev/mmcblk0p2 crypt_sdcard
# mkfs.ext4 /dev/mapper/crypt_sdcard
```

Restore rootfs to the encrypted volume and close the disk:
```
# mkdir -p /mnt/encrypted
# mount /dev/mapper/crypt_sdcard /mnt/encrypted/
# rsync -avh /mnt/backup/* /mnt/encrypted/
# umount /mnt/encrypted/
# cryptsetup luksClose /dev/mapper/crypt_sdcard
# sync
```

Eject SD card and test it on the Raspberry. When boot process
asks for passphrase, ssh to the box and enter the passphrase:
```
$ ssh -i kali-dropbear root@192.168.88.133
```

Once it is working, do not forget to clean up backup files on the host:
```
# rm -fr /mnt/backup
# rm -fr /mnt/chroot
```
