# Clone Linux system installed on hardware for QEMU
 
This howto is about how to clone Linux system installed on hardware for QEMU virtual
machine. I have tried this method on Arch Linux x86 64 and I think it can be workable
for other Linux distributions after little adaptation. You will need 2 Linux systems.
First is installed on hardware. This system which we will clone. Second is reserve
system written on USB-stick or CD/DVD-disk. I use `Arch Linux monthly iso`, which can
be downloaded from [Arch Linux web site][]. With help of second system we will clone first.

First of all you should boot your "hardware system" (I use this term for conciseness.
More correct it is called system which is installed on your physical equipment).
Although you can use reserve system for most steps, I prefer full-featured hardware
system. It is more comfortable. I guess that you have already installed QEMU and known
how to work with it.

## Step 1. Create QEMU raw disk image.
You should calculate size of all files and directories which belong your hardware
system, e.g., with `df` utility, and set appropriate QEMU image size according to it.

```
		$ df -h
		Filesystem      Size  Used Avail Use% Mounted on
		/dev/sda5       9.8G  1.5G  7.8G  17% /
		/dev/sda7        49G  7.3G   40G  16% /usr
		/dev/sda3       976M   97M  813M  11% /boot
		/dev/sda4       4.9G  934M  3.7G  20% /home
		/dev/sda6       4.9G   27M  4.6G   1% /tmp
		/dev/sda9       140G   26G  107G  20% /data
		/dev/sda8        15G  3.2G   12G  21% /var
```

My hardware system has 7 partitions with data. Only 5 of them contain system data which
will be clone. This is `sda3`, `sda4`, `sda5`, `sda7` and `sda8` partitions. `sda6` and
`sda9` partitions contain temporary system data and user data files correspondingly.
We will not clone it, but will create empty partitions for /tmp and /data
directories into QEMU disk image. I have calculated size of this image for my system,
but you should do it for yours.

* First I should notice that I will use GPT and legacy BIOS booting on QEMU virtual
machine so I need reserve 1 MiB for BIOS boot partition.
* In the second turn I going to isolate 1 GiB for `swap` partition.
* `/boot` directory uses 97 MiB on `sda3` partition. Let's say I plan to install couple
of additional kernels on my QEMU virtual machine and I think that 512 MiB will be enough
for this needs.
* `/home` directory uses 934 MiB on `sda4` partition. I plan to use `/home` only for
user configs, not for keeping data so I suppose that 2 GiB will be enough.
* Root directory (`/`) uses little bit more then `/home`, but again I plan to use it
only for configs so 2 GiB will be also enough. 
* `/tmp` directory uses very small disk space on my hardware system. I think that on
virtual machine it will occupy just as little space. So 1 GiB will be more then enough.
* `/usr` directory uses 7.3 GiB on `sda7` partition. It contains most of programs
installed on system. I don't plan to install much more. 10 GiB is enough for my
purpose. If you are going to install a lot of huge programs on your virtual machine
system you should make it bigger.
* `/var` directory use 3.2 GiB. It contains system logs and mails, schedule tasks,
package manager cache and other thing that should be cleared form time to time.
I plan to reserve 4 GiB for it. If you are going to use your virtual machine as server
you maybe will want to make it bigger.
* You can allocate arbitrary amount of space for /data directory. I will 1 GiB.

Let's sum up. `1 MiB` for BIOS boot + `1GiB` for swap + `512` MiB for /boot + `2 GiB`
for /home + `2 GiB` for /root + `1 GiB` for /tmp + `10 GiB` for /usr + `4 GiB` for
/var + `1 GiB` for /data approximately equal to 22 GiB.

```
		$ qemu-img create -f raw arch_snapshot.raw 22G
```

You can use `dd` utility instead of `qemu-img` for the same purpose, but it is slow.

```
		$ dd if=/dev/zero of=arch_snapshot.raw bs=1M count=22528 status=progress
```

Here 22 GiB (22528 * 1 MiB) file is created.

If your scheme of system partitions differs from mine, e.g., you have only one big
partition mounted to root (`/`) directory, you should calculate size of QEMU
disk image relying on how much space your system uses now and will use in future
on virtual machine.

## Step 2.
Now we need to create partition table and partitions into QEMU disk image.
I suggest to make them the same as on your hardware. You can see it with `fdisk`.
```
		# fdisk -l /dev/sda
		Disk /dev/sda: 232.9 GiB, 250059350016 bytes, 488397168 sectors
		...
		Disklabel type: gpt
		...
		Device         Start       End   Sectors   Size Type
		/dev/sda1       2048      4095      2048     1M BIOS boot
		/dev/sda2       4096   8392703   8388608     4G Linux swap
		/dev/sda3    8392704  10489855   2097152     1G Linux filesystem
		/dev/sda4   10489856  20975615  10485760     5G Linux filesystem
		/dev/sda5   20975616  41947135  20971520    10G Linux filesystem
		/dev/sda6   41947136  52432895  10485760     5G Linux filesystem
		/dev/sda7   52432896 157290495 104857600    50G Linux filesystem
		/dev/sda8  157290496 188747775  31457280    15G Linux filesystem
		/dev/sda9  188747776 488397134 299649359 142.9G Linux filesystem
```
Hard drive disk where my system is installed is `sda`. My partition table is `GPT` and
I have 9 partitions. `sda1` is BIOS boot partition, `sda2` is swap. `sda3-sda5`, `sda7`
and `sda8` contain Linux system files which we will copy later. `sda6` and `sda9`
contain temporary and user data files correspondingly, we will not copy them.

I going to create partition table and partitions according to this scheme. My partition
list is following (we have already calculated size of each partition on `Step 1`)
```
		partition 	size		type
		1. bios_boot 	1Mib		BIOS boot
		2. swap		1Gib		Linux swap
		3. boot		500Mib		Linux filesystem
		4. home		2Gib		Linux filesystem
		5. root		2Gib		Linux filesystem
		6. tmp		1Gib		Linux filesystem
		7. usr		10Gib		Linux filesystem
		8. var		4Gib		Linux filesystem
		9. data		1Gib		Linux filesystem
```

Your list will depends on partition scheme of media where Linux system is installed.
You can omitt partition for swap if your virtual machine will not require a lot of
memory. Also you can omitt separate partition for user data if you plan to place all in `/home`.
To create partition table and partitions use `fdisk` (you can find how to do it at
[fdisk]:[]) or other disk utility which you like.
```
		$ fdisk arch_snapshot.raw
```

## Step 3. Attach loopback device to QEMU disk image.
This device will allow us to mount into the system partitions created on `Step 2`.
Assumed that you have `loop` kernel module which is configured to allow attach loopback
devices with enough amount of partitions (9 in my case).
```
		# losetup -f -P arch_snapshot.raw
```

You can see loopback device has been created with `lsblk`
```
		$ lsblk -o NAME
		NAME
		loop0                                                               
		|-loop0p1                                                           
		|-loop0p2                                                           
		|-loop0p3                                                           
		|-loop0p4                                                           
		|-loop0p5                                                           
		|-loop0p6                                                           
		|-loop0p7                                                           
		|-loop0p8                                                           
		`-loop0p9                                                           
```
In my case it is device `loop0` with 9 partitions. `loop0p1` is BIOS boot partition,
`loop0p2` is for swap, `loop0p6` is for temporary system files and 
`loop0p3-loop0p5, loop0p7, loop0p8` are for Linux system files. Your loopback scheme
depends on free loopback devices names and partition scheme which you used when have
created partitions on `Step 3`.

## Step 4. Format loopback device partitions to appropriate file systems.
I suggest to make them the same as on your hardware. Use `lsblk` to see it.
```
		$ lsblk -o NAME,FSTYPE /dev/sda
		NAME      FSTYPE
		sda       
		|-sda1    
		|-sda2    swap
		|-sda3    ext4
		|-sda4    ext4
		|-sda5    ext4
		|-sda6    ext4
		|-sda7    ext4
		|-sda8    reiserfs
		`-sda9    ext4
```

My `sda3-sda7` and `sda9` partitions has `ext4` file system, `sda8` has `reiserfs`. So let's make the same on loopback device partitions.
```
		# mkfs.ext4 /dev/loop0p3
		# mkfs.ext4 /dev/loop0p4
		# mkfs.ext4 /dev/loop0p5
		# mkfs.ext4 /dev/loop0p6
		# mkfs.ext4 /dev/loop0p7
		# mkfs.reiserfs /dev/loop0p8
		# mkfs.ext4 /dev/loop0p9
```

Your file systems can be differ from mine, but again I recommend use the same as on
media where Linux system is installed.

## Step 5. Make swap on loopback swap partition.
Partition `loop0p2` in my case.
```
		# mkswap /dev/loop0p2
```
On next step we will directly clone our hardware system to QEMU disk image. Before
proceeding you should boot reserve Linux system from prepared bootable media.
Although you can do the following step from the same Linux system which you try to
clone, it is not desirable because some files could be changed during the copying
process, that can lead to errors and even make system on virtual machine not bootable.
So I highly recommend to use reserve system.

## Step 6.
After booting reserve system attach loopback device to QEMU disk image again.
```
		# losetup -f -P arch_snapshot.raw
	See which device has been made this time with 'lsblk'
		$ lsblk -o NAME
		NAME
		loop2                                                         
		|-loop2p1                                                     
		|-loop2p2 
		|-loop2p3
		|-loop2p4
		|-loop2p5 
		|-loop2p6 
		|-loop2p7 
		|-loop2p8 
		`-loop2p9 
```

In my case it is `loop2` device with 9 partitions `loop2p1-loop2p9`. We will copy data
from hardware system partitions to loopback device partitions, so let's create
appropriate directories (mount points) for source and destination.
```
		# mkdir /mnt/source_part
		# mkdir /mnt/dest_part
```

Currently we can begin procedure of coping. Mount first partition with hardware system
data (`sda3` in my case) read only into source directory and corresponding loopback
device partition (`loop2p3` in my case) into destination directory. 
```
		# mount -o ro /dev/sda3 /mnt/source_part
		# mount /dev/loop2p3 /mnt/dest_part
```

Copy all files from source to destination.
```
		# cp -a /mnt/source_part/* /mnt/dest_part
```

Parameter `-a` for `cp` is important(!). Without it additional files attributes
(context, links, xattr, etc) will not be copied and virtual machine will login
correct only for root. Also it enables coping directories recursively.
Now unmount hardware system and loopback device partitions.

```
		# umount /dev/sda3
		# umount /dev/loop2p3
```

Repeat this procedure for every remaining partition (`sda4`, `sda5`, `sda7`, `sda8` in my case).

When you have finished leave reserve Linux system and boot again your hardware system.
On next step we will correct obtained clone-system in order to make it workable on QEMU
virtual machine. 

## Step 7.
After boot attach loopback device to QEMU disk image again.
```
		# losetup -f -P arch_snapshot.raw
```

Use `lsblk` to see which device has been created
```
		$ lsblk -f
		NAME      FSTYPE   LABEL UUID                                 MOUNTPOINT
		loop0                                                         
		|-loop0p1                                                     
		|-loop0p2 swap           7e5659eb-ab86-4781-b99b-9b5d9ed86099 
		|-loop0p3 ext4           761a058b-e869-4a0c-bdb7-e850bc850363 
		|-loop0p4 ext4           106971f2-b344-4453-8a10-d2a04429d495 
		|-loop0p5 ext4           e0541c81-014b-4182-92cc-f2aca1ff4e88 
		|-loop0p6 ext4           c4b59c10-e007-4c57-b7b5-7f27689c0860 
		|-loop0p7 ext4           62602ff9-0525-4eba-bf0d-85bdec0cc073 
		|-loop0p8 reiserfs       b136dfd4-5072-4fb7-9405-9177bada2f66 
		`-loop0p9 ext4           1a6180f4-da2c-4acc-9cec-773250489ad7 
```
It is `loop0` device in my case again. Now partitions of loopback device contain Linux
cloned system, but it isn't even bootable. We should make correction in some
configuration files and install bootloader. For this purpose mounting of cloned
partitions is required. We will mount them similar to they will be mounted on virtual
machine. Let's create directory (mount point) for root (/) cloned partition and mount
it.
```
		# mkdir /mnt/chroot
		# mount /dev/loop0p5 /mnt/chroot 
```

I have named directory `chroot` and it will be clear later why. My cloned root
partition is `loop0p5`. Now we should mount other partitions. Mount points already
exist in `/mnt/chroot` directory.
```
		# mount /dev/loop0p3 /mnt/chroot/boot
		# mount /dev/loop0p4 /mnt/chroot/home
		# mount /dev/loop0p6 /mnt/chroot/tmp
		# mount /dev/loop0p7 /mnt/chroot/usr
		# mount /dev/loop0p8 /mnt/chroot/var
		# mount /dev/loop0p9 /mnt/chroot/data
```

Next is most interesting. Change apparent root directory to `/mnt/chroot` with
`arch-chroot` for Arch Linux or another utility wich your distribution grants.
```
		# arch-chroot /mnt/chroot
```

Correct `/boot/grub/grub.cfg` and `/etc/fstab` according to your cloned partition
scheme. How to make config in these files beyond the scope of this
howto. You can find some information in Arch Wiki pages [GRUB][] and [Fstab][].
There are some recommendations:
* If you use UUID based config then correct all fields which contain UUID of
partitions. You can know new UUIDs from `lsblk -f` command listing (see above).
* Delete or comment fstab legacy entries which declare partitions that don't exist on
cloned system.
* If you don't use UUID and have saved partition scheme of hardware system don't change
anything. Any way system informs you what is wrong on boot process.

And last but not least install GRUB bootloader on your cloned system.
```
		# grub-install --target=i386-pc /dev/loop0
```
Now you can exit chroot environment.
```
		$ exit
```
Voila! Now you can boot your cloned system in `fallback initramfs` mode on QEMU virtual
machine, but first unmount all cloned partitions and detach loopback device.
```
		# umount /dev/loop0p3
		# umount /dev/loop0p4
		# umount /dev/loop0p6
		# umount /dev/loop0p7
		# umount /dev/loop0p8
		# umount /dev/loop0p9
		# umount /dev/loop0p5
		# losetup -d /dev/loop0
```

## Step 8.
Start virtual machine with embedded QEMU raw disk image and boot the clone-system in
`fallback initramfs` mode (choose appropriate item in GRUB menu).
```
		$ qemu-system-x86_64 -enable-kvm -m 2G -vga virtio -display gtk,gl=on -drive file=arch_snapshot.raw,format=raw
```

After boot create new `initramfs image` with `mkinitcpio` for Arch Linux or another
utility which your distribution grants.
```
		# mkinitcpio -p linux
```

After that restart virtual machine.

[Arch Linux web site]: https://www.archlinux.org/download/
[fdisk]: https://wiki.archlinux.org/index.php/Fdisk
[GRUB]: https://wiki.archlinux.org/index.php/GRUB
[Fstab]: https://wiki.archlinux.org/index.php/Fstab
