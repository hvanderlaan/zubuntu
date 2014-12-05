These instructions are for Ubuntu.  The procedure for Debian, Mint, or other distributions in the DEB family is similar but not identical.

### System Requirements
  * 64-bit Ubuntu Live CD.  (Not the alternate installer, and not the 32-bit installer!)
  * AMD64 or EM64T compatible computer. (ie: x86-64)
  * 8GB disk storage available.
  * 2GB memory minimum.

Computers that have less than 2GB of memory run ZFS slowly.  4GB of memory is recommended for normal performance in basic workloads.  16GB of memory is the recommended minimum for deduplication.  Enabling deduplication is a permanent change that cannot be easily reverted.

### Latest Tested And Recommended Version
* Ubuntu 14.04 Trusty Tahr
* spl-0.6.3
* zfs-0.6.3

## Step 1: Prepare The Install Environment

1.1 Start the Ubuntu LiveCD and open a terminal at the desktop.

1.2 Input these commands at the terminal prompt:

	$ sudo -i
	# apt-add-repository --yes ppa:zfs-native/stable
	# apt-get update
	# apt-get install debootstrap spl-dkms zfs-dkms ubuntu-zfs

1.3 Check that the ZFS filesystem is installed and available:

	# modprobe zfs
	# dmesg | grep ZFS:
	ZFS: Loaded module v0.6.3-2~trusty, ZFS pool version 5000, ZFS filesystem version 5


## Step 2: Disk Partitioning

This tutorial intentionally recommends MBR partitioning.  GPT can be used instead, but beware of UEFI firmware bugs.

2.1 Run your favorite disk partitioner, like `parted` or `cfdisk`, on the primary storage device.  `/dev/disk/by-id/scsi-SATA_disk1` is the example device used in this document.

2.2 Create a small MBR primary partition of **at least** 8 megabytes.  256mb may be more realistic, unless space is tight.  `/dev/disk/by-id/scsi-SATA_disk1-part1` is the example boot partition used in this document.

2.3 On this first small partition, set `type=BE` and enable the `bootable` flag.

2.4 Create a large partition of **at least** 4 gigabytes.  `/dev/disk/by-id/scsi-SATA_disk1-part2` is the example system partition used in this document.

2.5 On this second large partition, set `type=BF` and disable the `bootable` flag.

The partition table should look like this:

	# fdisk -l /dev/disk/by-id/scsi-SATA_disk1
	
	Disk /dev/sda: 10.7 GB, 10737418240 bytes
	255 heads, 63 sectors/track, 1305 cylinders
	Units = cylinders of 16065 * 512 = 8225280 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disk identifier: 0x00000000
	
	Device    Boot      Start         End      Blocks   Id  System
	/dev/sda1    *          1           1        8001   be  Solaris boot
	/dev/sda2               2        1305    10474380   bf  Solaris

**Remember:**  Substitute `scsi-SATA_disk1-part1` and `scsi-SATA_disk1-part2` appropriately below.

**Hints:**
* Are you doing this in a virtual machine?  Is something in `/dev/disk/by-id` missing?  Go read the troubleshooting section.
*  Recent GRUB releases assume that the `/boot/grub/grubenv` file is writable by the stage2 module.  Until GRUB gets a ZFS write enhancement, the GRUB modules should be installed to a separate filesystem in a separate partition that is grub-writable.
* If `/boot/grub` is in the ZFS filesystem, then GRUB will fail to boot with this message:  `error: sparse file not allowed`.  If you absolutely want only one filesystem, then remove the call to `recordfail()` in each `grub.cfg` menu stanza, and edit the `/etc/grub.d/10_linux` file to make the change permanent.
* Alternatively, if `/boot/grub` is in the ZFS filesystem you can comment each line with the text `save_env` in the file `/etc/grub.d/00_header` and run update-grub.  


## Step 3: Disk Formatting

3.1  Format the small boot partition created by Step 2.2 as a filesystem that has stage1 GRUB support like this:

	# mke2fs -m 0 -L /boot/grub -j /dev/disk/by-id/scsi-SATA_disk1-part1

3.2  Create the root pool on the larger partition:

	# zpool create -o ashift=9 rpool /dev/disk/by-id/scsi-SATA_disk1-part2

Always use the long /dev/disk/by-id/* aliases with ZFS.  Using the /dev/sd* device nodes directly can cause sporadic import failures, especially on systems that have more than one storage pool.

**Warning:** The grub2-1.99 package currently published in the PPA for Precise does not reliably handle a 4k block size, which is `ashift=12`.

**Hints:**
* `# ls -la /dev/disk/by-id` will list the aliases.
* The root pool can be a mirror.  For example, `zpool create -o ashift=9 rpool mirror /dev/disk/by-id/scsi-SATA_disk1-part2 /dev/disk/by-id/scsi-SATA_disk2-part2`. Remember that the version and ashift matter for any pool that GRUB must read, and that these things are difficult to change after pool creation.
* If you are using a mirror with a separate boot partition as described above, don't forget to edit the grub.cfg file <b>on the second HD partition</b> so that the "root=" partition refers to that partition on the second HD also; otherwise, if you lose the first disk, you won't be able to boot from the second because the kernel will keep trying to mount the root partition from the first disk.
* The pool name is arbitrary.  On systems that can automatically install to ZFS, the root pool is named "rpool" by default.  Note that system recovery is easier if you choose a unique name instead of "rpool".  Anything except "rpool" or "tank", like the hostname, would be a good choice.
* If you want to create a mirror but only have one disk available now you can create the mirror using a sparse file as the second member then immediately off-line it so the mirror is in degraded mode.  Later you can add another drive to the spool and ZFS will automatically sync them.  The sparse file won't take up more than a few KB so it can be bigger than your running system.  Just make sure to off-line the sparse file before writing to the pool.  

3.2.1 Create a sparse file at least as big as the larger partition on your HDD:

	# truncate -s 11g /tmp/sparsefile

3.2.2 Instead of the command in section 3.2 use this to create the mirror:

	# zpool create -o ashift=9 rpool mirror /dev/disk/by-id/scsi-SATA_disk1-part2 /tmp/sparsefile

3.2.3 Offline the sparse file.  You can delete it after this if you want.

	# zpool offline rpool /tmp/sparsefile

3.2.4 Verify that the pool was created and is now degraded.

	# zpool list
	NAME       SIZE  ALLOC   FREE    CAP  DEDUP  HEALTH  ALTROOT
	rpool     10.5G   188K  10.5G     0%  1.00x  DEGRADED  -


3.3  Create a "ROOT" filesystem in the root pool:

	# zfs create rpool/ROOT

3.4  Create a descendant filesystem for the Ubuntu system:

	# zfs create rpool/ROOT/ubuntu-1

On Solaris systems, the root filesystem is cloned and the suffix is incremented for major system changes through `pkg image-update` or `beadm`.  Similar functionality for APT is possible but currently unimplemented.

3.5  Dismount all ZFS filesystems.

	# zfs umount -a

3.6  Set the `mountpoint` property on the root filesystem:

	# zfs set mountpoint=/ rpool/ROOT/ubuntu-1

3.7  Set the `bootfs` property on the root pool.

	# zpool set bootfs=rpool/ROOT/ubuntu-1 rpool

The boot loader uses these two properties to find and start the operating system.  These property names are *not* arbitrary.

**Hint:**  Putting `rpool=MyPool` or `bootfs=MyPool/ROOT/system-1` on the kernel command line overrides the ZFS properties.

3.9  Export the pool:

	# zpool export rpool

Don't skip this step.  The system is put into an inconsistent state if this command fails or if you reboot at this point.


## Step 4:  System Installation

**Remember:** Substitute "rpool" for the name chosen in Step 3.2.

4.1  Import the pool:

	# zpool import -d /dev/disk/by-id -R /mnt rpool

If this fails with "cannot import 'rpool': no such pool available", you can try import the pool without the device name eg:

        # zpool import -R /mnt rpool

4.2  Mount the small boot filesystem for GRUB that was created in step 3.1: 

	# mkdir -p /mnt/boot/grub
	# mount /dev/disk/by-id/scsi-SATA_disk1-part1 /mnt/boot/grub

4.4  Install the minimal system:

	# debootstrap trusty /mnt

The `debootstrap` command leaves the new system in an unconfigured state.  In Step 5, we will only do the minimum amount of configuration necessary to make the new system runnable.


## Step 5:  System Configuration

5.1  Copy these files from the LiveCD environment to the new system:

	# cp /etc/hostname /mnt/etc/
	# cp /etc/hosts /mnt/etc/

5.2  The `/mnt/etc/fstab` file should be empty except for a comment.  Add this line to the `/mnt/etc/fstab` file:

	/dev/disk/by-id/scsi-SATA_disk1-part1  /boot/grub  auto  defaults  0  1

The regular Ubuntu desktop installer may add `dev`, `proc`, `sys`, or `tmp` lines to the `/etc/fstab` file, but such entries are redundant on a system that has a `/lib/init/fstab` file.  Add them now if you want them.

5.3  Edit the `/mnt/etc/network/interfaces` file so that it contains something like this:

	# interfaces(5) file used by ifup(8) and ifdown(8)
	auto lo
	iface lo inet loopback
	
	auto eth0
	iface eth0 inet dhcp

Customize this file if the new system is not a DHCP client on the LAN.

5.4  Make virtual filesystems in the LiveCD environment visible to the new system and `chroot` into it:

	# mount --bind /dev  /mnt/dev
	# mount --bind /proc /mnt/proc
	# mount --bind /sys  /mnt/sys
	# chroot /mnt /bin/bash --login

5.5 Install PPA support in the chroot environment like this:

	# locale-gen en_US.UTF-8
	# apt-get update
	# apt-get install ubuntu-minimal software-properties-common

Even if you prefer a non-English system language, always ensure that en_US.UTF-8 is available.  The ubuntu-minimal package is required to use ZoL as packaged in the PPA.

5.6  Install ZFS in the chroot environment for the new system:

	# apt-add-repository --yes ppa:zfs-native/stable
	# apt-add-repository --yes ppa:zfs-native/grub
	# apt-get update
	# apt-get install --no-install-recommends linux-image-generic linux-headers-generic
	# apt-get install ubuntu-zfs
	# apt-get install grub2-common grub-pc
	# apt-get install zfs-initramfs
	# apt-get dist-upgrade

**Warning:** This is the second time that you must wait for the SPL and ZFS modules to compile.  *Do not* try to skip this step by copying anything from the host environment into the chroot environment.

**Note:** This should install a kernel package and its headers, a patched mountall and dkms packages.  Double-check that you are getting these packages from the PPA if you are deviating from these instructions in any way.

Choose `/dev/disk/by-id/scsi-SATA_disk1` if prompted to install the MBR loader.

Ignore warnings that are caused by the chroot environment like:

* `Can not write log, openpty() failed (/dev/pts not mounted?)`
* `df: Warning: cannot read table of mounted file systems`
* `mtab is not present at /etc/mtab.`

5.7  Set a root password on the new system:

	# passwd root

**Hint:** If you want the ubuntu-desktop package, then install it after the first reboot.  If you install it now, then it will start several process that must be manually stopped before dismount.


## Step 6:  GRUB Installation

**Remember:** All of Step 6 depends on Step 5.4 and must happen inside the chroot environment.

6.1  Verify that the ZFS root filesystem is recognized by GRUB:

	# grub-probe /
	zfs

And that the ZFS modules for GRUB are installed:

	# ls /boot/grub/zfs*
	/boot/grub/zfs.mod  /boot/grub/zfsinfo.mod

Note that after Ubuntu 13, these are now in /boot/grub/i386/pc/zfs*

	# ls /boot/grub/i386-pc/zfs*
	/boot/grub/i386-pc/zfs.mod  /boot/grub/i386-pc/zfsinfo.mod

Otherwise, check the troubleshooting notes for GRUB below.

6.2  Refresh the initrd files:

	# update-initramfs -c -k all
	update-initramfs: Generating /boot/initrd.img-3.2.0-40-generic

6.3  Update the boot configuration file:

	# update-grub
	Generating grub.cfg ...
	Found linux image: /boot/vmlinuz-3.2.0-40-generic
	Found initrd image: /boot/initrd.img-3.2.0-40-generic
	done

Verify that `boot=zfs` appears in the boot configuration file:

	# grep boot=zfs /boot/grub/grub.cfg
	linux /ROOT/ubuntu-1/@/boot/vmlinuz-3.2.0-40-generic root=/dev/sda2 ro boot=zfs $bootfs quiet splash $vt_handoff
	linux /ROOT/ubuntu-1/@/boot/vmlinuz-3.2.0-40-generic root=/dev/sda2 ro single nomodeset boot=zfs $bootfs


6.4  Install the boot loader to the MBR like this:

	# grub-install $(readlink -f /dev/disk/by-id/scsi-SATA_disk1)
	Installation finished. No error reported.

Do not reboot the computer until you get exactly that result message. Note that you are installing the loader to the whole disk, not a partition.

**Note:** The readlink is required because recent GRUB releases do not dereference symlinks.


## Step 7:  Cleanup and First Reboot

7.1  Exit from the `chroot` environment back to the LiveCD environment:

	# exit

7.2  Run these commands in the LiveCD environment to dismount all filesystems:

	# umount /mnt/boot/grub
	# umount /mnt/dev
	# umount /mnt/proc
	# umount /mnt/sys
	# zfs umount -a
	# zpool export rpool

The `zpool export` command must succeed without being forced or the new system will fail to start.

7.3  We're done!

	# reboot


## Caveats and Known Problems

### This is an experimental system configuration.

This document was first published in 2010 to demonstrate that the `lzfs` implementation made ZoL 0.5 feature complete.  Upstream integration efforts began in 2012, and it will be at least a few more years before this kind of configuration is even minimally supported.

Gentoo, and its derivatives, are the only Linux distributions that are currently mainlining support for a ZoL root filesystem.

### `zpool.cache` inconsistencies cause random pool import failures.

The `/etc/zfs/zpool.cache` file embedded in the initrd for each kernel image must be the same as the `/etc/zfs/zpool.cache` file in the regular system.  Run `update-initramfs -c -k all` after any `/sbin/zpool` command changes the `/etc/zfs/zpool.cache` file.

Pools do not show up in `/etc/zfs/zpool.cache` when imported with the `-R` flag.

This will be a recurring problem until issue [zfsonlinux/zfs#330](https://github.com/zfsonlinux/zfs/issues/330) is resolved.

### Every upgrade can break the system.

Ubuntu systems remove old dkms modules before installing new dkms modules.  If the system crashes or restarts during a ZoL module upgrade, which is a failure window of several minutes, then the system becomes unbootable and must be rescued. 

This will be a recurring problem until issue [zfsonlinux/pkg-zfs#12](https://github.com/zfsonlinux/pkg-zfs/issues/12) is resolved.

When doing an upgrade remotely an extra precaution would be to use screen, this way if you get disconnected your installation will not get interrupted.

# Troubleshooting

## (i) MPT2SAS

Most problem reports for this tutorial involve `mpt2sas` hardware that does slow asynchronous drive initialization, like some IBM M1015 or OEM-branded cards that have been flashed to the reference LSI firmware.

The basic problem is that disks on these controllers are not visible to the Linux kernel until after the regular system is started, and ZoL does not hotplug pool members.  See https://github.com/zfsonlinux/zfs/issues/330.

Most LSI cards are perfectly compatible with ZoL, but there is no known fix if your card has this glitch.  Please use different equipment until the `mpt2sas` incompatibility is diagnosed and fixed, or donate an affected part if you want solution sooner.

## (ii) Areca

Systems that require the `arcsas` blob driver should add it to the `/etc/initramfs-tools/modules` file and run `update-initramfs -c -k all`.

Upgrade or downgrade the Areca driver if something like `RIP: 0010:[<ffffffff8101b316>]  [<ffffffff8101b316>] native_read_tsc+0x6/0x20` appears anywhere in kernel log.  ZoL is unstable on systems that emit this error message.


## (iii) GRUB Installation

Verify that the PPA for the ZFS enhanced GRUB is installed:

	# apt-add-repository ppa:zfs-native/grub
	# apt-get update

Reinstall the `zfs-grub` package, which is an alias for a patched `grub-common` package:

	# apt-get install --reinstall zfs-grub

Afterwards, this should happen:

	# apt-cache search zfs-grub
	grub-common - GRand Unified Bootloader (common files)
	
	# apt-cache show zfs-grub
	N: Can't select versions from package 'zfs-grub' as it is purely virtual
	N: No packages found
	
	# apt-cache policy grub-common zfs-grub
	grub-common:
	 Installed: 1.99-21ubuntu3.9+zfs1~precise1
	 Candidate: 1.99-21ubuntu3.9+zfs1~precise1
	 Version table:
	*** 1.99-21ubuntu3.9+zfs1~precise1 0
	      1001 http://ppa.launchpad.net/zfs-native/grub/ubuntu/precise/main amd64 Packages
	       100 /var/lib/dpkg/status
	    1.99-21ubuntu3 0
	      1001 http://us.archive.ubuntu.com/ubuntu/ precise/main amd64 Packages
	zfs-grub:
	 Installed: (none)
	 Candidate: (none)
	 Version table:

For safety, grub modules are never updated by the packaging system after initial installation. Manually refresh them by doing this:

	# cp /usr/lib/grub/i386-pc/*.mod /boot/grub/

If the problem persists, then open a bug report and attach the entire output of those `apt-get` commands.

Packages in the GRUB PPA are compiled against the stable PPA.  Systems that run the daily PPA may experience failures if the ZoL library interface changes.

Note that GRUB does not currently dereference symbolic links in a ZFS filesystem, so you cannot use the  `/vmlinux` or `/initrd.img` symlinks as GRUB command arguments.

## (iv) GRUB does not support ZFS Compression

If the `/boot` hierarchy is in ZFS, then that pool should not be compressed.  The grub packages for Ubuntu are usually incapable of loading a kernel image or initrd from a compressed dataset.

## (v) VMware

* Set `disk.EnableUUID = "TRUE"` in the vmx file or vsphere configuration.  Doing this ensures that `/dev/disk` aliases are created in the guest.

## (vi) QEMU/KVM/XEN

* In the `/etc/default/grub` file,  enable the `GRUB_TERMINAL=console` line and remove the `splash` option from the `GRUB_CMDLINE_LINUX_DEFAULT` line.  Plymouth can cause boot errors in these virtual environments that are difficult to diagnose.

* Set a unique serial number on each virtual disk.  (eg: `-drive if=none,id=disk1,file=disk1.qcow2,serial=1234567890`)

## (vii) Kernel Parameters

The `zfs-initramfs` package requires that `boot=zfs` always be on the kernel command line.  If the `boot=zfs` parameter is not set, then the init process skips the ZFS routine entirely.  This behavior is for safety; it makes the casual installation of the zfs-initramfs package unlikely to break a working system.

ZFS properties can be overridden on the the kernel command line with `rpool` and `bootfs` arguments.  For example, at the GRUB prompt:

`linux /ROOT/ubuntu-1/@/boot/vmlinuz-3.0.0-15-generic boot=zfs rpool=AltPool bootfs=AltPool/ROOT/foobar-3`

## (viii) System Recovery

If the system randomly fails to import the root filesystem pool, then do this at the initramfs recovery prompt:

	# zpool export rpool
	: now export all other pools too
	# zpool import -d /dev/disk/by-id -f -N rpool
	: now import all other pools too
	# mount -t zfs -o zfsutil rpool/ROOT/ubuntu-1 /root
	: do not mount any other filesystem
	# cp /etc/zfs/zpool.cache /root/etc/zfs/zpool.cache
	# exit

This refreshes the `/etc/zfs/zpool.cache` file.  The `zpool` command emits spurious error messages regarding missing or corrupt vdevs if the `zpool.cache` file is stale or otherwise incorrect.
