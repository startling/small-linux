## make image

````sh
qemu-img create -f raw boots.img 4G
````

# format and partition

`fdisk boots.img`
type `n` and then go with the defaults to make a new primary 1st partition taking the whole disk.
type `a` and then `1` to make partition 1 bootable.

# mount partitions

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