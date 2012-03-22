# a small linux (with qemu and Busybox)

Assumptions:

* you're running linux and have the necessary build tools and qemu installed
* you're comfortable with the command line
* you've got about half an hour to spare

I'm running this on arch, but I've tried to be distro-agnostic; still, you may have to improvise the necessary incantations. If you find anything confusing or wrong, feel free to email me: <tdixon51793@gmail.com>.

Sounds good? Let's go.

## Making the Image

`qemu-img` is a tool for making disk images that ships with `qemu`; we'll use it here to make a 4GB raw image file.

````sh
qemu-img create -f raw boots.img 4G
````

### Format and Partition

We'll use `fdisk` to format the image.

`fdisk boots.img`
type `n` and then go with the defaults to make a new primary first partition taking the whole disk.
type `a` and then `1` to make partition the first partition bootable. finally, type `w` to write to the disk image and quit.
### Mounting the Partition

We're using two slightly obscure tools here -- `losetup`, which creates what fake block devices from files, and `kpartx` (which you may need to install) which creates device files for each partition of a device.

````sh
# set up a loop device
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
* the `-append` options tell qemu to pass some startup options to the kernel; this helps us since we're not using a bootloader yet. The first of these, `root=/dev/sda1` says to use the first partition of the first drive (the one we formatted before).
* The second startup option, `console=ttyS0` says to use the serial port as the console. This handily lets us use the `-nographic` option and get the prompt right in our terminal.

Anyway, this is the kind of output you should get:

````
[    6.581884] Kernel panic - not syncing: No init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.
[    6.592880] Pid: 1, comm: swapper/0 Not tainted 3.2.12 #1
[    6.597796] Call Trace:
[    6.600915]  [<c15ff2f7>] ? printk+0x18/0x1a
[    6.605454]  [<c15ff1e9>] panic+0x57/0x14d
[    6.609289]  [<c15fe0cc>] init_post+0xa9/0xa9
[    6.613390]  [<c186d843>] kernel_init+0x136/0x136
[    6.617659]  [<c186d70d>] ? start_kernel+0x2e1/0x2e1
[    6.621881]  [<c160e0f6>] kernel_thread_helper+0x6/0xd
````

Our first kernel panic! How cute! You should take a picture.

## Calming Down The Kernel

First of all, the only real way to quit qemu after a kernel panic i've found is `killall qemu` or `killall qemu-system-i386` in another shell. Seems drastic, I know, but we'll resurrect the poor little guy in a moment.

So, what went wrong? Well, if you're familiar with the boot process (if you aren't, _[From PowerUp To Bash Prompt][]_ is pretty good), the kernel starts up, gets everything ready, and then hands off control to a binary named `init`. It couldn't find such a binary (perhaps because none exists yet? more research is needed) and so puked all over the kitchen floor. Fun.

[From PowerUp to Bash Prompt]: http://www.linuxdoc.org/HOWTO/From-PowerUp-To-Bash-Prompt-HOWTO-6.html

### Busybox

Now, we could install [sysvinit][] and [GNU Coreutils][] and bash or something to get started, essentially taking a page out of [Linux From Scratch's instructions][]. But that's a pain and we have more important things to learn than `./configure; make; make install`. Instead, we'll install [Busybox][] -- a single tiny executable that can be called as [a thousand different things][] -- `ash` and `init` and `which` and `wget` and so on -- and we'll get our system self-hosting _after_ we have it booting.

[sysvinit]: https://savannah.nongnu.org/projects/sysvinit
[GNU Coreutils]: http://www.gnu.org/software/coreutils/
[Busybox]: http://www.busybox.net/
[Linux From Scratch's instructions]: http://www.linuxfromscratch.org/lfs/view/stable/
[a thousand different things]: http://www.busybox.net/downloads/BusyBox.html

So download Busybox, untar/unzip it, and `cd` in:

````sh
wget http://www.busybox.net/downloads/busybox-1.19.4.tar.bz2
tar xjf busybox-1.19.4.tar.bz2
cd busybox-1.19.4/
````

and poke around in `make menuconfig`; when you're done, exit and save your configuration.

> __Side note__: the thing to do here would be to configure Busybox to compile statically. I can't get it to do that, though; my compilation fails with a
> 
> ```
> collect2: ld returned 1 exit status
> make: *** [busybox_unstripped] Error 1
> ````
> 
> Because of this, i'm going to need to do a silly lazy hack later. If you want to try compiling statically yourself, though, feel free to -- the menu option is `Busybox settings -->`, `Build Options -->`, `Build Busybox as a static binary`.

And then `make` and, assuming your directory structure is the same as mine:

````
# mount our image again
sudo mount /dev/mapper/loop0p1 ../disk
# and then install Busybox to it
sudo make CONFIG_PREFIX=../disk install
````

You'll see an exciting swirl of activity (Busybox's `make install` politely tells you about all the symlinks it makes, so you can notice if it missed one) and then we're done.

### Ugly Lazy Hacks

Things are going to start getting exciting real quick.

First, though, the ugly lazy hack because I couldn't get Busybox to compile statically:

* go back into the mounted directory -- `cd ../disk`
* look to see what dynamic libraries Busybox needs with `ldd bin/busybox`. I get:
    
    ````
        linux-gate.so.1 =>  (0xb76df000)
        libm.so.6 => /lib/libm.so.6 (0xb76a9000)
        libc.so.6 => /lib/libc.so.6 (0xb7507000)
        /lib/ld-linux.so.2 (0xb76e0000)
    ````

    The first of these (`linux-gate.so.1`) is what's know as a 'virtual shared dynamic object'. Basically, it's a fake library provided by the kernel. We can ignore it -- the rest are real dynamic libraries.

* This is the lazy part. We *could* build a libc ourselves for the system; instead, we'll copy the dynamic libraries we need from our host system:
    
    ````
    cp /lib/libm.so.6 lib/
    cp /lib/libc.so.6 lib/
    cp /lib/ld-linux.so.2 lib/
    ````

### Diving In

And then, finally, we can chroot in and see how things look. `sudo chroot . /bin/ash`. What you see now is the prompt for the `ash` shell that's part of Busybox. Poke around until you're content that we almost have a real system.

But wait! There are a few things we still need. 

> I think it's neat to do all this with the Busybox `vi` in the chroot, but you can exit with ctrl-d and do them with whatever text editor you want. Wooh misplaced enthusiasm!

First, make these directories:

`mkdir etc boot dev proc sys tmp`

#### /etc/passwd

We'll need an `/etc/passwd`, with some junk in it because we don't feel like hand-hashing a password.

````
root:butts:0:0:root:/:/bin/sh
````

and then run `passwd` (while `chroot`ed) to put an actual hashed password there.

#### /etc/group
`/etc/group` is actually pretty boring.

````
root:x:0:
````

#### /etc/fstab

Our `/etc/fstab` can be real simple since we only have one partition:

````
# /etc/fstab: static file system information                                                           
# <file system> <dir>   <type>  <options>   <dump>  <pass>
tmpfs /tmp tmpfs nodev,nosuid 0 0
````

We don't mount the root partition here becase we'll do it in our start script.


#### /etc/inittab

Inittabs for Busybox are a little unconventional -- they completely ignore runlevels and use the `id` field for controlling devices (defaulting to /dev/console). We'll do two main things here: start a script we'll write in a moment (`/etc/start`) and spawn a handful of ttys.

````
# simple start script
::sysinit:/etc/start

# getty for login over console/the serial port
::respawn:/sbin/getty -L ttyS0 9600 linux

# gettys for tty1-4
tty1::respawn:/sbin/getty 38400 tty1
tty2::respawn:/sbin/getty 38400 tty2
tty3::respawn:/sbin/getty 38400 tty3
tty4::respawn:/sbin/getty 38400 tty4

# stuff to do before rebooting
::ctrlaltdel:/bin/umount -a -r
::ctrlaltdel:/sbin/swapoff

# adapted from http://www.spblinux.de/2.0/doc/init.html
````

#### /etc/start

A lot of these guides have tedious little interludes where they either use devfs (which is deprecated and dead) or or static device files made with `mknod`. We don't have to do that; Busybox comes with this great little utility called `mdev`. You can read more about it in the `docs/mdev.txt` in the Busybox source.

````
#!/bin/sh

# mount /proc
mount -t proc proc /proc
# mount /sys
mount -t sysfs sysfs /sys
# set mdev as the kernel's hotplug for dynamic updaes
echo /sbin/mdev > /proc/sys/kernel/hotplug
# mount all the things in /etc/fstab
mount -s
# remount the root partition as read-write
mount -o rw,remount /
# mount devices with mdev
mdev -s
````

Afterwards, `sudo chmod +x etc/start`, too, in order to make it executable.

## Booting For Real

So. Here's the moment of truth. Unmount the disk (`sudo umount /dev/mapper/loop0p1`, but you already knew that) and fire up qemu again.

Again, on Arch:

`qemu-system-i386 -kernel bzImage -hda boots.img -append "root=/dev/sda1 console=ttyS0" -nographic`

and on most other distros:

`qemu -kernel bzImage -hda boots.img -append "root=/dev/sda1 console=ttyS0" -nographic`

Hopefully you'll be greeted with a thunderstorm of kernel activity and then a prompt like `(none) login`. Enter `root` and the password you set earlier and you're home free.

(P.S. -- Busybox doesn't have `shutdown -h`; instead, use `poweroff` once you're done.)

## Cleaning Up

There are a few minor annoyances here:

### Kernel and Bootloader

(todo)

### Networking

(todo)

### hostname

The reason the login prompt is `(none) login` is that we haven't set a hostname for the system. `hostname boots` or whatever will set the hostname for one session; put it in your `/etc/start` for it to persist.


## Build Tools

(todo)

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
