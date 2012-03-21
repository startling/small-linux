# a small linux (with qemu and busybox)

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
sudo losetup -f boots.img // maybe different outside of arch? research
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

## Linux

First thing to do is download the linux kernel source:

`wget https://www.kernel.org/pub/linux/kernel/v3.0/linux-3.2.12.tar.bz2`

and then untar/unbzip it and cd in:

````sh
tar xjf linux-3.2.12.tar.bz2
cd linux-3.2.12/
````

Building the linux kernel from source can be slightly complicated; luckily, the default options work fine for us. You can `make menuconfig` and poke around a little bit to see what choices we have, and then exit and save your configuration. `make all` builds the entire thing; go and make some tea or something while you wait for it.


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