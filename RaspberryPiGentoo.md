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

Alternatively, you can get this information using the `file` command:

    $ file 2013-05-25-wheezy-raspbian.img

Then, take the value of `startsector` for partition 2, and multiply by 512, you'll get the same value.

You can then mount the / partition, as root:

    $ mkdir /mnt/raspberry
    $ mount -o offset=62914560 /mnt/raspberry

You need to mount it because one module which is set to be loaded at boot time will prevent you to log in:

    $ nano /mnt/raspberry/etc/ld.so.preload # comment out the loaded module (with `#`)
    $ umount /mnt/raspberry

I expected to copy the kernel from the boot partition and start from here, but that kernel wont boot, and I have no idea why. You can find a specific kernel that will work on qemu [here](http://xecdesign.com/downloads/linux-qemu/kernel-qemu). Put is inside the same folder as your disk image, and you should be ready to go.

You can then start the machine. Note that:

* You need to specify the kernel specific to qemu
* You can't change the allocated memory.
* Run the following command:

    $ qemu-system-arm -kernel kernel-qemu -cpu arm1176 -m 256 -M versatilepb -no-reboot -serial stdio -append "root=/dev/sda2 panic=1" -hda 2013-05-25-wheezy-raspbian.img

The first time you boot, you will get prompted to check your drive, run:

    $ fsck /dev/sda2

Then reboot:

    $ shutdown -r now

To retart the machine, use the `qemu` command given above. You can also launch with `--curses` if your don't want to start a window or if you're through ssh.

The second time you boot, you'll be inside a config tool, you can use it for instance to change the password (user is `pi`, default password is `raspberry`) or start the ssh server.

The `pi` user can sudo. The network should be usable, although I can't ping hosts. Your first command could be to update and upgrade your packages:

    $ sudo apt-get update
    $ sudo apt-get upgrade

If you started qemu with an X window, you can start the dektop manager:

    $ startx


# 2. Issues

* In the QEMU window, some characters are not correctly mapped (such as #), no matter you're in the console or on LXDE. 
* You can't ping hosts, although it resolves names correctly and other network software work well.

# 3. References

<http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/>
<http://www.raspberrypi.org/phpBB3/viewtopic.php?f=29&t=37386>