Emulating a Raspberry Pi on Gentoo
==================================

This document describes how to develop and experiment with the Raspberry Pi using a Gentoo desktop or server. We want to show how to emulate a Raspberry device on Gentoo and also cross-compile for the Raspberry from a standard Gentoo instance. We do not intend to install Gentoo on the Raspberry but focus on using the well-optimized Raspbian distribution.

# A. Emulating a Raspberry device on Gentoo

Using qemu, one can emulate the Raspberry hardware and run OSes compiled for it. We are going to use [Rapsbian](http://www.raspbian.org/), based on Debian, which seems to be the most supported OS for the Raspberry. Other possibilities include Arch Linux and Gentoo, of course.

## 1. Procedure

On Gentoo, install `app-emulation/qemu` with the `qemu_softmmu_targets_arm` USE flag. There are a few kernel parameters that need to be set, you'll get warned when you install it.

On the Raspberry website, disk images of Raspbian are available. The standard way of using them on the Raspberry hardware is to write them on a SD card, and use the SD card directly as hard drive. As of today, the latest image is `2013-05-25-wheezy-raspbian.zip`.

In the archive, there is one `img` file, which itself contains two partitions. You can mount, but you need to know the offset value for that. Using `parted`, that's quite easy:

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

The offset is the start sector of the second partition  62914560. Alternatively, you can get this information using the `file` command:

    $ file 2013-05-25-wheezy-raspbian.img

Then, take the value of `startsector` for partition 2, and multiply by 512, you'll get the same value.

You can then mount the / partition, as root:

    $ mkdir /mnt/raspberry
    $ mount -o offset=62914560 /mnt/raspberry

You need to mount it because one module which is set to be loaded at boot time will prevent you to log in (we don't use the same kernel as the one used when the module was compiled):

    $ nano /mnt/raspberry/etc/ld.so.preload # comment out the loaded module (with `#`)
    $ umount /mnt/raspberry

I expected to copy the kernel from the boot partition and start from here, but that kernel won't boot, and I have no idea why. You can find a specific kernel that will work on qemu [here](http://xecdesign.com/downloads/linux-qemu/kernel-qemu). Put is inside the same folder as your disk image, and you should be ready to go.

You can then start the machine. Note that:

* You need to specify the kernel specific to qemu.
* You can't change the allocated memory.
* Run the following command:

    $ qemu-system-arm -kernel kernel-qemu -cpu arm1176 -m 256 -M versatilepb -no-reboot -serial stdio -append "root=/dev/sda2 panic=1" -hda 2013-05-25-wheezy-raspbian.img

The first time you boot, you will get prompted to check your drive, run:

    $ fsck /dev/sda2

Then reboot:

    $ shutdown -r now

To restart the machine, use the `qemu` command given above. You can also launch with `--curses` if you don't want to start a window or if you're through ssh.

The second time you boot, you'll be inside a config tool, you can use it for instance to change the password (user is `pi`, default password is `raspberry`) or start the ssh server.

The `pi` user can sudo, and the network should be usable. Your first command could be to update and upgrade your packages:

    $ sudo apt-get update
    $ sudo apt-get upgrade

If you started qemu with an X window, you can start the desktop manager:

    $ startx

If you want to `ssh` or `scp` to your emulated device, you can start qemu with `-redir tcp:5022::22`, which redirects port 5022 of the host to port 22 (ssh port) on the guest. This is an unprivileged port, it does not require you to run qemu as root.

you can for example:

    $ scp -P 5022 myfile pi@myhost:~/
    $ ssh -p 5022 pi@myhost

Note the capital `-P` for scp and lowercase `-p` for ssh. If you're already on the host, `myhost` can be `localhost`.

## 2. Issues

* In the qemu window, some characters are not correctly mapped (such as # and @, which is very annoying), no matter you're in the console or on LXDE. 
* You can't ping hosts, although it resolves names correctly and other network software work well.
* The first time the image starts, you will be required to run `fsck`, then reboot the emulated device. It seems that the first boot can not happen in an environment without X (with qemu started with `-curses` or `-nographic`), but upcoming startups work fine in this context. If you want to run your emulator on a distant machine, through ssh, one possibility is to do the first boot on a machine that has X available, then copy the image to the remote server.

## 3. References

<http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/>
<http://www.raspberrypi.org/phpBB3/viewtopic.php?f=29&t=37386>

# B. Cross compilation

The goal here is to be able to compile software directly on a Gentoo desktop that is binary compatible with the Raspberry device and OS. Doing so, development is more comfortable than on the resource limited-device or qemu emulated-device and probably much faster.

## 1. Procedure

Gentoo has a well-documented cross development handbook, we summarize the procedure here:

You first need to install `sys-devel/crossdev`, you should probably use the latest version by adding `~amd64` keyword, or the one that matches your architecture.

Then you need to install the toolchain, but before that, you need some adjustments on your system:

Create `/usr/local/portage`

    $ mkdir /usr/local/portage

Add `PORTDIR_OVERLAY="/usr/local/portage"` to `/etc/make.conf`.

Switch to platform-specific `package.xxx` files:

    $ su
    $ mv /etc/portage/package.keywords ~/
    $ mkdir -p /etc/portage/package.keywords/x86_64-pc-linux-gnu # if you're on x86_64
    $ mv ~/package.keywords /etc/portage/package.keywords/x86_64-pc-linux-gnu

And repeat this for `package.use` and `package.mask`.

You can then install the toolchain, as root:

    $ crossdev -S -t armv6j-hardfloat-linux-gnueabi

If successful, it will install `binutils`, `linux-headers`, `glibc` and `gcc`.

You can give it a try with:

    $ armv6j-hardfloat-linux-gnueabi-gcc --version

And try a small "hello world" program. Create `hello.cpp`:


    #include <iostream>
    int main() {
      std::cout << "Hello, World!\n";
      return 0;
    }

and compile it with:

    $ armv6j-hardfloat-linux-gnueabi-g++ hello.cpp -o hello

then copy it to your virtual raspberry:

    $ scp -P 5022 hello pi@localhost:~/

Your "hello world" program should run successfully.

## 2. References

<http://www.gentoo.org/proj/en/base/embedded/handbook/index.xml>
<http://wiki.gentoo.org/wiki/Raspberry_Pi>