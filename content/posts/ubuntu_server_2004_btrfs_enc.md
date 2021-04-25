---
title: "Encrypted BTRFS Root on Ubuntu Server 20.04"
date: 2021-04-10T17:27:30Z
draft: true
---

Ubuntu switched their server installer to [Subiquity](https://github.com/canonical/subiquity) with the release of Bionic
Beaver (18.04).
While it does have advantages over the older [Debian Installer("d-i")](https://wiki.debian.org/DebianInstaller/), it's
manual disk partitioner leaves something to be desired for those of us with more complex requirements.
It used to be possible to use the `mini.iso` installer to get an image using the Debian Installer, but that is no longer
officially supported, and it is likely that that image will no longer be built at all in future releases.

The preferred method for customization seems to be their network installer and [automatic installation
config](https://ubuntu.com/server/docs/install/autoinstall) build on the cloud-init platform.  Still, custom LUKS
encrypted partitioning doesn't seem to be simple using this system - at least I couldn't figure it out quickly - and
doesn't seem worth the effort for small scale users.

So, without further ado, here is how to install Ubuntu 20.04 server with a LUKS encrypted BTRFS (or any filesystem of
your choice) with the Subiquity installer.


## Requirements

* Ubuntu 20.04.x Server bootable media (or any image using Subiquity)

* ~3G writeable space outside of root

    This can be the USB drive you've booted on, extra space in RAM available during the install, or extra swap space.


## Installing

For the general server installation procedure, I recommend you use Canonical's [Ubuntu Server installation
guide](https://ubuntu.com/server/docs/installation), or one of the many other guides available.


For this, I'll be using a VM image with 20G of disk and 4GB ram and a UEFI bootloader.


### Step 0: Create Partitions

This is the only step you are probably going to want to do _during_ the main installation process, as it lets the
installer write all the required files to the correct partitions, so you don't have to think as much about where files
need to go later.

![Custom Paritions](../images/ubuntu_server_2004_btrfs_enc/installer_07_05_final.png)


Here I've gotten all my partitions set up for a UEFI boot.
Remember that when installing under UEFI, the Ubuntu partitioner will create `/boot/efi` partition for you when you
create the base `/boot` partition.


### Step 1: Jump to the End

Now all that's left is to let the installer do it's work.

Once the installation has "completed," we need to enter the shell to actually perform the encryption.

![We're Done?](../images/ubuntu_server_2004_btrfs_enc/installer_12_enter_shell.png)

#### Aside

If you jump into the shell immediately, you may notice that all the partitions seem to be in place right after the
partitioning step.

```bash
root@ubuntu-server:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           394M  1.3M  392M   1% /run
/dev/sr0        1.2G  1.2G     0 100% /cdrom
/dev/loop0      357M  357M     0 100% /rofs
/cow            2.0G   36M  1.9G   2% /
/dev/loop2       60M   60M     0 100% /usr/lib/modules
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs           2.0G   28K  2.0G   1% /tmp
/dev/loop3       56M   56M     0 100% /snap/core18/1944
/dev/loop4       70M   70M     0 100% /snap/lxd/19188
/dev/loop5       32M   32M     0 100% /snap/snapd/10707
/dev/loop6       53M   53M     0 100% /snap/subiquity/2280
tmpfs           394M     0  394M   0% /run/user/1000
/dev/vda4        15G 1017M   14G   7% /target
/dev/vda2       488M  3.1M  450M   1% /target/boot
/dev/vda1       511M  4.0K  511M   1% /target/boot/efi
overlay         2.0G   28K  2.0G   1% /target/etc/apt
```

However, you may also notice the `overlay` filesystem mounted at `/target/etc/apt`.
Until this is gone, the installer may still be installing requirements, so we should wait.


### Step 2: Back up the Root

If you run `df` immediately, you'll see that the our partitions are all mounted under `/target`.

```shell
root@ubuntu-server:/# df -hlPT
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  1.9G     0  1.9G   0% /dev
tmpfs          tmpfs     394M  1.3M  392M   1% /run
/dev/sr0       iso9660   1.2G  1.2G     0 100% /cdrom
/dev/loop0     squashfs  357M  357M     0 100% /rofs
/cow           overlay   2.0G   36M  1.9G   2% /
/dev/loop2     squashfs   60M   60M     0 100% /usr/lib/modules
tmpfs          tmpfs     2.0G     0  2.0G   0% /dev/shm
tmpfs          tmpfs     5.0M     0  5.0M   0% /run/lock
tmpfs          tmpfs     2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs          tmpfs     2.0G   32K  2.0G   1% /tmp
/dev/loop3     squashfs   56M   56M     0 100% /snap/core18/1944
/dev/loop4     squashfs   70M   70M     0 100% /snap/lxd/19188
/dev/loop5     squashfs   32M   32M     0 100% /snap/snapd/10707
/dev/loop6     squashfs   53M   53M     0 100% /snap/subiquity/2280
tmpfs          tmpfs     394M     0  394M   0% /run/user/1000
/dev/vda4      btrfs      15G  2.1G   13G  15% /target
/dev/vda2      ext4      488M  107M  346M  24% /target/boot
/dev/vda1      vfat      511M  7.9M  504M   2% /target/boot/efi
```

First, unmount the `boot` and `efi` partitions, and create a directory to store the root data, which I'll call `/tgt`
here:

```bash
umount /target/boot/efi
umount /target/boot
mkdir /tgt
```

Now, we need to copy the data on `/target` (which will be `/`) to `/tgt` for safekeeping while we encrypt and format the
partition.
However, on this system, we only have 4GB of RAM, which is split between `/dev` and `/` on the installation system, and
we don't have any room on the installation media since we are installing from an ISO.
If you have enough room on `/` or the installation media (just mount it at `/tgt`), you can skip past this step.


#### Reformat SWAP temporarily

We have an extra 4GB SWAP partition (`/dev/vda3`) that we can reuse - we just have to reformat it and mount it at `/tgt`
so we can access it:

```bash
root@ubuntu-server:/# mkfs.ext4 /dev/vda3
mke2fs 1.45.5 (07-Jan-2020)
/dev/vda3 contains a swap file system
Proceed anyway? (y,N) y
Discarding device blocks: done                            
Creating filesystem with 1048576 4k blocks and 262144 inodes
Filesystem UUID: 52b784b9-ee7c-47ff-8d43-1ef561ef7c0d
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

root@ubuntu-server:/# mount /dev/vda3 /tgt
```

#### Copy Root Files

It's very important to make sure you *maintain permissions* on all of the files you are copying over.
Failing to do so can render your system unusable, or at least highly insecure, once you have finished.
Using the `-a/--archive` flag with `cp` will make sure that all the permissions are maintained.

```bash
cp -a /target/. /tgt/
```

Be aware that you might need more space than is listed by `df` if left your root formatted as BTRFS (like I did).

```bash
root@ubuntu-server:/# du -sh /tgt/
3.2G	/tgt/
```


### Step 3: Encrypt Root and Format to BTRFS

For us, our root partition is a `/dev/vda4`, and is still mounted at `/target`, so we need to unmount it and then set up
the encryption:

```bash
root@ubuntu-server:/# umount -l /target

root@ubuntu-server:/# cryptsetup luksFormat /dev/vda4
WARNING: Device /dev/vda4 already contains a 'btrfs' superblock signature.

WARNING!
========
This will overwrite data on /dev/vda4 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/vda4: 
Verify passphrase: 

root@ubuntu-server:/# cryptsetup open /dev/vda4 cryptroot
Enter passphrase for /dev/vda4: 

root@ubuntu-server:/# mkfs.btrfs /dev/mapper/cryptroot
btrfs-progs v5.4.1 
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               d1aab86f-8592-4f7e-8bf9-e634cd021bd9
Node size:          16384
Sector size:        4096
Filesystem size:    14.98GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             256.00MiB
  System:           DUP               8.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1    14.98GiB  /dev/mapper/cryptroot
```

### Step 4: Prepare BTRFS Subvolumes

Now, you can set up your BTRFS however you like.
I am going to create four subvolumes for:

* `/`
* `/home`
* `/var/log`
* `/.snapshots`


```bash
root@ubuntu-server:/# mkdir /broot

root@ubuntu-server:/# mount -t btrfs /dev/mapper/cryptroot /broot/

root@ubuntu-server:/# btrfs subvolume create /broot/@
Create subvolume '/broot/@'

root@ubuntu-server:/# btrfs subvolume create /broot/@home
Create subvolume '/broot/@home'

root@ubuntu-server:/# btrfs subvolume create @snapshots
Create subvolume '/broot/@snapshots'

root@ubuntu-server:/# mkdir /broot/@/var

root@ubuntu-server:/# btrfs subvolume create /broot/@/var/log
Create subvolume '/broot/@/var/log'

root@ubuntu-server:/# btrfs subvolume list /broot
ID 257 gen 10 top level 5 path @
ID 258 gen 8 top level 5 path @home
ID 259 gen 9 top level 5 path @snapshots
ID 260 gen 10 top level 257 path @/var/log
```

### Step 5: Remount As `/target`

We need to set `/target` to be the mount point for our new root, and then copy our installation files back into that
directory.

```bash
root@ubuntu-server:/# mount -t btrfs -o noatime,subvol=@ /dev/mapper/cryptroot /target

root@ubuntu-server:/# mkdir /target/home

root@ubuntu-server:/# mount -t btrfs -o noatime,subvol=@home /dev/mapper/cryptroot /target
mount: /target: /dev/mapper/cryptroot already mounted on /broot.

root@ubuntu-server:/# mount -t btrfs -o noatime,subvol=@home /dev/mapper/cryptroot /target/home

root@ubuntu-server:/# cp -a /tgt/. /target/

root@ubuntu-server:/# mount /dev/vda2 /target/boot

root@ubuntu-server:/# mount /dev/vda1 /target/boot/efi
```


#### Unmount and Remake swap Partion

```bash
root@ubuntu-server:/# umount /broot/

root@ubuntu-server:/# umount /tgt

root@ubuntu-server:/# mkswap /dev/vda3
mkswap: /dev/vda3: warning: wiping old ext4 signature.
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
no label, UUID=9b39065e-c6ed-4cd9-ba33-150d9f301b02
```


### Step 6: `chroot` to Finish Configuration

```bash
root@ubuntu-server:/# mount --bind /dev /target/dev

root@ubuntu-server:/# mount --bind /proc /target/proc

root@ubuntu-server:/# mount --bind /sys /target/sys

root@ubuntu-server:/# chroot /target

root@ubuntu-server:/# blkid /dev/vda* 
/dev/vda: PTUUID="a264fe1b-687e-4425-bbca-be1043359ac2" PTTYPE="gpt"
/dev/vda1: UUID="0C93-7305" TYPE="vfat" PARTUUID="3f7ef7f1-4fb9-4db2-a971-40e140643112"
/dev/vda2: UUID="b0c09191-fb22-4a5c-939a-f088526a8b34" TYPE="ext4" PARTUUID="b7acde71-1a05-48d3-b2a3-7368907d510b"
/dev/vda3: UUID="9b39065e-c6ed-4cd9-ba33-150d9f301b02" TYPE="swap" PARTUUID="a27cd316-1f88-4800-86b6-3aed5aa3e6ed"
/dev/vda4: UUID="4ab86ab2-14f8-42d9-bf07-33894a9d7807" TYPE="crypto_LUKS" PARTUUID="04f6597b-4d80-4394-a89c-89007aeb3199"
```

#### Add Encrypted Partition to `/etc/crypttab`


```conf
# <target name> <source device>         <key file>      <options>
cryptroot /dev/disk/by-uuid/4ab86ab2-14f8-42d9-bf07-33894a9d7807 none luks,discard
```


#### Edit `/etc/fstab`


* update swap
* not mounting `@snapshots` yet

```conf
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/disk/by-uuid/9b39065e-c6ed-4cd9-ba33-150d9f301b02 none swap sw 0 0
# / was on /dev/vda4 during curtin installation
/dev/mapper/cryptroot / btrfs defaults,noatime,subvol=@ 0 0
/dev/mapper/cryptroot /home btrfs defaults,noatime,subvol=@home 0 0
# /boot was on /dev/vda3 during curtin installation
/dev/disk/by-uuid/b0c09191-fb22-4a5c-939a-f088526a8b34 /boot ext4 defaults 0 0
# /boot/efi was on /dev/vda1 during curtin installation
/dev/disk/by-uuid/0C93-7305 /boot/efi vfat defaults 0 0
```

#### Step 5: Update `initramfs` and grub

```bash
root@ubuntu-server:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-5.8.0-50-generic

root@ubuntu-server:/# update-grub        
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/init-select.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.8.0-50-generic
Found initrd image: /boot/initrd.img-5.8.0-50-generic
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
logger: socket /dev/log: Connection refused
  WARNING: Device /dev/dm-0 not initialized in udev database even after waiting 10000000 microseconds.
  WARNING: Device /dev/dm-0 not initialized in udev database even after waiting 10000000 microseconds.
logger: socket /dev/log: Connection refused
Adding boot menu entry for UEFI Firmware Settings
done
```

### Step 6: Exit chroot and Reboot

```bash
root@ubuntu-server:/# exit
exit

root@ubuntu-server:/# reboot -p now
```


### Step 7: Unlock and Enjoy!
