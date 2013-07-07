Emulating a Raspberry Pi on Gentoo
==================================

This document describes how to develop and experiment with the Raspberry Pi using a Gentoo desktop. Eventually, we want to show how to emulate a Raspberry device on Gentoo and also cross-compile for the Rasperry from a standard Gentoo desktop.

# 1. Emulating a Raspberry device on Gentoo

Using QEMU, one can emulate the Raspberry hardware and run OSes compiled for it. We are going to use [Rapsbian](http://www.raspbian.org/), based on Debian, which seems to be the most supported OS for the Raspberry. Other possibilities include Arch Linux and Gentoo, of course.

On Gentoo, install `app-emulation/qemu` with the `qemu_softmmu_targets_arm` USE flag. There are a few kernel parameters that need to be set, you'll get warned when you install it.

On the Rasberry website, disk images of Raspbian are available. The standard way of using them on the Raspberry hardware is to write them on a SD card, and use the SD card directly as hard drive. As of today, the latest image is `2013-05-25-wheezy-raspbian.zip`.

In the archive, there is one `img` file, which itself contains two partitions. You can mount them with mount, but you need to know the offset value for that. Using `parted`, that's quite easy:

    $ su # We need to do these things as root
    $ parted 2013-05-25-wheezy-raspbian.img
    ...
    (parted) unit
    Unit?  [compact]? B # To get values in Bytes
    (parted) print
    Model:  (file)
    Disk /home/sylvain/raspbian/2013-05-25-wheezy-raspbian.img: 1939865600B
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:
    Number  Start      End          Size         Type     File system  Flags
    1      4194304B   62914559B    58720256B    primary  fat16        lba
    2      62914560B  1939865599B  1876951040B  primary  ext4

    (parted) quit

You can then mount the / partition, as root:

    $ mkdir /mnt/raspberry
    $ mount -o offset=62914560 /mnt/raspberry

# 2. References

<http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/>
<http://www.raspberrypi.org/phpBB3/viewtopic.php?f=29&t=37386>