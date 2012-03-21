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

## references:

### QEMU
* Allan Stephen's _[QEMU Cheat Sheet][]_ on the linuxkernelnewbies mailing list.

### Linux
* _[How To Build a Minimal Linux System from Source Code][]_ by Greg O'Keefe

### Busybox

[QEMU Cheat Sheet]: http://www.mail-archive.com/linuxkernelnewbies@googlegroups.com/msg00826.html
[How To Build a Minimal Linux System from Source Code]: http://users.cecs.anu.edu.au/~okeefe/p2b/buildMin/buildMin.html