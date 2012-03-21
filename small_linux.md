# a small linux (with qemu and busybox)

Assumptions:

* you're running linux and have the necessary build tools and qemu installed
* you're comfortable with the command line
* you've got about half an hour to spare

I'm running this on arch, but I've tried to be distro-agnostic; still, you may have to improvise the necessary incantations. If you find anything confusing or wrong, feel free to email me: <tdixon51793@gmail.com>.

Sounds good? Let's go.

## make image

````sh
qemu-img create -f raw boots.img 4G
````

## format and partition

`fdisk boots.img`
type `n` and then go with the defaults to make a new primary first partition taking the whole disk.
type `a` and then `1` to make partition the first partition bootable. finally, type `w` to write to the disk image and quit.

## mount partitions

````sh
# set up a loopback device
sudo losetup -f boots.img # maybe different outside of arch? research
# kpartx the partitions
sudo kpartx -a /dev/loop0
# there should now exist a /dev/mapper/loop0p1 or similar
# make an ex3 filesystem:
sudo mkfs.ext3 /dev/mapper/loop0p1
# and, finally, mount our partition
mkdir disk
sudo mount /dev/mapper/loop0p1 disk
````

you can `ls disk` now and see that it contains a single directory, `lost+found`, which is made when you format as ext.

## Building Linux

First thing to do is download the linux kernel source:

`wget https://www.kernel.org/pub/linux/kernel/v3.0/linux-3.2.12.tar.bz2`

and then untar/unbzip it and cd in:

````sh
tar xjf linux-3.2.12.tar.bz2
cd linux-3.2.12/
````

Building the linux kernel from source can be slightly complicated; luckily, the default options work fine for us. You can `make menuconfig` and poke around a little bit to see what choices we have, and then exit and save your configuration. `make all` builds the entire thing; go and make some tea or something while you wait for it.

Once you're done, copy the image to the directory _above_ the mounted image for now:

`cp arch/x86/boot/bzImage ../`

and install the kernel headers on the disk, too:

`sudo make INSTALL_MOD_PATH=../disk modules_install`

Now `cd ..` and survey all that we have made.

## Figuring Out How To Boot

We're going to use qemu to look into booting are disk real quick. Unmount the disk image first (`sudo umount /dev/mapper/loop0p1`), and then choose one of the following:

If you're on Arch Linux,

`qemu-system-i386 -kernel bzImage -hda boots.img -append "root=/dev/sda1 console=ttyS0" -nographic`.

Otherwise,

`qemu -kernel bzImage -hda boots.img -append "root=/dev/sda1 console=ttyS0" -nographic`.

This looks a little imposing, so let's break it up a little:

* The `-kernel bzImage` tells qemu to use the kernel we compiled. Note that this is outside of the image.
* `-hda boots.img` tells qemu that our image file is the system's first hard drive. no big deal.
* the `-append` options tell qemu to pass some startup options to the kernel. `root=/dev/sda1` says to use the first partition of the first drive (the one we formatted before); `console=ttyS0` says to use the serial port as the console. This handily lets us use the `-nographic` option and get the prompt right in our terminal.

## references:

* Allan Stephen's _[QEMU Cheat Sheet][]_ on the linuxkernelnewbies mailing list.
* _[How To Build a Minimal Linux System from Source Code][]_ by Greg O'Keefe
* _[Building Tiny Linux Systems With Busybox][]_ by Bruce Perens
* _[Building a Minimal Linux Image][]_ on the University of Maine High Performance Computing Wiki.
* _[Booting/Building a Minimal Busybox based Linux distro][]_

[QEMU Cheat Sheet]: http://www.mail-archive.com/linuxkernelnewbies@googlegroups.com/msg00826.html
[How To Build a Minimal Linux System from Source Code]: http://users.cecs.anu.edu.au/~okeefe/p2b/buildMin/buildMin.html
[Building Tiny Linux Systems With Busybox]: http://www.linuxjournal.com/article/4335
[Building a Minimal Linux Image]: http://www.clusters.umaine.edu/wiki/index.php/Building_a_Minimal_Linux_Image
[Booting/Building a Minimal Busybox based Linux distro]: https://revcode.wordpress.com/2012/02/25/booting-a-minimal-busybox-based-linux-distro/
