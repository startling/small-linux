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

Now, we could install [sysvinit][] and [GNU Coreutils][] and bash or something to get started, essentially taking a page out of [Linux From Scratch's instructions][]. But that's a pain and we have more important things to learn than `./configure; make; make install`. Instead, we'll install [busybox][] -- a single tiny executable that can be called as [a thousand different things][] -- `ash` and `init` and `which` and `wget` and so on -- and we'll get our system self-hosting _after_ we have it booting.

[sysvinit]: https://savannah.nongnu.org/projects/sysvinit
[GNU Coreutils]: http://www.gnu.org/software/coreutils/
[busybox]: http://www.busybox.net/
[Linux From Scratch's instructions]: http://www.linuxfromscratch.org/lfs/view/stable/
[a thousand different things]: http://www.busybox.net/downloads/BusyBox.html

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
