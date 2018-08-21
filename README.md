# Fedora 28 using root ZFS, LUKS, UEFI, and GPT disks

Do you like Fedora, but want to use ZFS as your root filesystem? Do you want to encrypt that ZFS filesystem with LUKS? Do you want to also boot your system using UEFI and GPT disks? You can combine multiple guides that already exist and guess your way to completion...or you can follow this guide where I've already done all those things for you. If you're of the TL;DR sort, feel free to skip straight to the [Installation](#installation) section.

## Assumptions
This guide will:
- Use Linux. Hopefully that's obvious, but in case you wondered this guide won't be about installing software on your favorite headgear.
- Use GPT partition tables. That seems pretty specific, but GPT does not have as much coverage in other guides as MBR tables do.
- Use UEFI to boot the system
  - By extension, this guide will use a system with a BIOS that supports UEFI booting
  - Yes, "legacy" booting is still very possible but it also seems very well-covered by other guides.
- Use Fedora 28. Older versions aren't covered, and newer versions could always break things.
- Use ZFS packages from [ZFS On Linux](https://zfsonlinux.org/), not the FUSE packages that come with Fedora
- Use an unencrypted `/boot` partition (it is theoretically possible to encrypt `/boot`, but it's outside the scope of this guide)
- Use only a single hard drive for simplicity, even though ZFS is most beneficial when used across several hard drives
- NOT require extra flash drives or other removable media, just the Fedora 28 install media
- NOT specifically ask you to unlock your drives after every reboot. If you see the LUKS prompt, enter your passphrase and carry on
- Use ZFS On Linux's automatic `ashift` detection rather than specifying a value. The `ashift` value most commonly depends upon whether your disks are "Advanced Format" (4k sector size) or not. If you care to really check on this, a link to trying to find whether your disks are "Advanced Format" can be found at the end of this guide

## Conventions
- `/dev/sdx` will be the notation used for the hard drive's path. Replace it with your actual path wherever it appears. (The actual path convention for your system will differ depending on OS, distribution, hardware, and other potential variables.)
  - **Example:** `/dev/sdx1` ("the first partition in `/dev/sdx`")
- Anything prefaced with a "$" sign and in ALL_CAPS_WITH_UNDERSCORES is a variable, and something you're supposed to fill in with an actual value
  - **Example:** `$THIS_VALUE`
- Wherever possible, things that should be typed or executed all at once are enclosed in code blocks while individual commands will have individual bullet points or list items
- Unless specifically mentioned, all commands _are executed as the root user_. We're playing with filesystems and installing an OS, using `root` is not only warranted but sometimes the only way.

## Partition Layout
The following partition layout is what we will setup in the beginning of this guide. Note that `/boot` and `/boot/efi` remain constant throughout the process and are simply overwritten or appended to after we switch from initial installation to having ZFS as the root filesystem.
- `/dev/sdx1` (Mounts at "/boot/efi")
- `/dev/sdx2` (Mounts at "/boot")
- `/dev/sdx3` (Initially does not mount, later becomes "/")
- `/dev/sdx4` (Initially mounts at "/", later is deleted and becomes part of `/dev/sdx3`)

## Installation
1. Begin by inserting a USB stick or DVD containing Fedora 28's Live installer into your computer
1. Boot the system in UEFI mode (how exactly you do this will depend on your system)
1. From the boot selection screen, select the "Start Fedora-workstation-live ..." option
1. Press the "e" key to edit the boot options.
   - **NOTE:** If the instructions say to press "tab" instead, stop because you're using legacy BIOS booting and not UEFI
1. Add "inst.gpt" to end of the line that begins with `vmlinuz`
   - **Example:** `vmlinuz ... quiet inst.gpt`
   - This is just a precaution to force the use of GPT partition tables, as the Fedora live media doesn't automatically use GPT tables on disks smaller than a certain size
1. Use the keyboard shortcut `Ctrl-x` to start the edited boot entry
1. When asked whether you wish to try Fedora or install to the hard drive, select "Try"
1. Open a terminal in the Fedora desktop and use `sudo -i` to become the root user. (From here on, it's assumed you are running all commands as `root` unless otherwise specified.)
1. Use the partition program "parted" to delete existing partitions and initialize a new GPT partition table by running the command `parted /dev/sdx mklabel gpt`
1. Use your favorite partition utility (as long as it supports GPT partition tables) to create four partitions:
   - 256MB partition (type is "EFI System Partition")
   - 512MB partition (type is "Linux filesystem")
   - Partition A that fills half the rest of disk (type is "Solaris /usr")
   - Partition B that fills the rest of disk (type is "Linux filesystem")
1. Save the new partition table and exit your partition utility
1. To determine the "best" crypto algorithm for LUKS, run the command `cryptsetup benchmark`
1. Note name of the algorithm which gives the best balance of key size, and encryption/decryption speed. In my case this was "aes-xts-plain64"
1. Encrypt both Partition A and B with LUKS by running the following commands (supply a new passphrase of your choosing when asked)
   - `cryptsetup luksFormat -c <algorithmName> -s <keysize> -h sha256 --iter-time 5000 --use-urandom /dev/sdx3`
   - `cryptsetup luksFormat -c <algorithmName> -s <keysize> -h sha256 --iter-time 5000 --use-urandom /dev/sdx4`
   - **NOTE:** This step can be performed in a few minutes by Fedora's partition installer, but you wouldn't necessarily get the same range of command line options
1. Press the "Meta" key (on a lot of keyboards, it's the "Windows" key), type `inst` and select "Install to Hard Drive" from Fedora's menu
1. Proceed with the Fedora installation as normal until arriving at the partitioning screen
1. Choose the "Custom" Storage Configuration and click "Done"
1. Choose "standard" (not "LVM") partitioning
1. Select and decrypt the `/dev/sdx3` and `/dev/sdx4` LUKS partitions through the GUI
1. Once the LUKS partitions are unlocked, use the "Reformat" checkbox on `/dev/sdx4` to select the "ext4" filesystem and a mountpoint of "/"
1. Use the "Reformat" checkbox on `/dev/sdx1` to select the "vfat" filesystem and a mountpoint of "/boot/efi"
1. Use the "Reformat" checkbox on `/dev/sdx2` to select the "ext3" filesystem and a mountpoint of "/boot"
1. At this point, the Fedora partition screen should basically have the following structure
   - /dev/sdx1 = /boot/efi
   - /dev/sdx2 = /boot
   - /dev/sdx4 = /
   - /dev/sdx3 = (none or "Unknown")
1. Anaconda should recognize that this is a UEFI system. Press "Done"
1. Check the warning that ensues at the bottom of the screen to make sure that it's _only_ warning about no swap space being defined. If there are other warnings, heed them. Otherwise, click "Done" again to move on.
1. In the terminal that's still open once the installer closes, make sure to set a password for the `root` user by running the command `passwd` and defining one
1. **REBOOT THE SYSTEM**
1. After reboot, wait for the login screen and use the keyboard shortcut `Ctrl-Alt-F2` to ignore it in favor of using the terminal
1. Login as `root` using the password created in Step 25
1. Run the command `lsblk` and note down the names of the decrypted LUKS volume that was not mounted by the Anaconda installer (it's under the `/dev/sdx3` heading). This is **IMPORTANT** much later, should be in the format "luks-<UUID>", and will be referred to later as `$LUKS_UUID`.
1. Update the system to the latest packages by running the command `dnf update -y` (if this says `waiting for pid...` that's normal, the metadata cache takes a few minutes to build)
1. Verify UEFI installation with the following two commands
   - `rpm -qa | grep grub2-efi`
   - `rpm -qa | grep shim`
1. If both packages are installed, continue. Otherwise, your system is likely not using UEFI and parts of this guide aren't likely to work
1. Run the command `dnf install grub2-efi-x64-modules` to pull in the `zfs` module
1. Create the following script called `zmogrify` in `/usr/local/sbin` (this can actually be anywhere in $PATH)) that will update dracut when new versions of the ZFS packages are installed in the future

    ```
    #!/bin/bash
    # zmogrify - Run dracut and configure grub to boot with root on zfs.
    
    kver=$1
    sh -x /usr/bin/dracut -fv --kver $kver
    mount /boot/efi
    grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
    grubby --set-default-index 0
    mkdir -p /boot/efi/EFI/fedora/x86_64-efi
    cp -a /usr/lib/grub/x86_64-efi/zfs* /boot/efi/EFI/fedora/x86_64-efi
    cp /usr/lib/grub/x86_64-efi/{procfs,cryptodisk,luks,crypto,gcry_*,archelp}.mod /boot/efi/EFI/fedora/x86_64-efi
    umount /boot/efi
    ```

1. Make the `zmogrify` script executable by running the command `chmod a+x zmogrify`
1. Create a script called `zenter` in `/usr/local/sbin` (again, anywhere on $PATH would do) that will automatically setup necessary mounts and enter a "chroot"

    ```
    #!/bin/bash
    # zenter - Mount system directories and enter a chroot
    
    target=$1
    
    mount -t proc  proc $target/proc
    mount -t sysfs sys $target/sys
    mount -o bind /dev $target/dev
    mount -o bind /dev/pts $target/dev/pts
    
    chroot $target /bin/env -i \
        HOME=/root TERM="$TERM" PS1='[\u@chroot \W]\$ ' \
        PATH=/bin:/usr/bin:/sbin:/usr/sbin:/bin \
        /bin/bash --login
    
    echo "Exiting chroot environment..."
    
    umount $target/dev/pts
    umount $target/dev/
    umount $target/sys/
    umount $target/proc/
    ```

1. Make the `zenter` script executable by running the command `chmod a+x zenter`
1. Using your text editor of choice, edit `/etc/default/grub` to enable ZFS support (later) by adding the line

    ```
    GRUB_PRELOAD_MODULES="zfs"
    ```

1. In order to work around a later bug in grub2-mkconfig that affects ZFS, create the file `/etc/profile.d/grub2_zpool_fix.sh"`and insert the following line

    ```
    export ZPOOL_VDEV_NAME_PATH=YES
    ```

1. Set Fedora to run `zmogrify` after each new kernel update by running the command `ln -s /usr/local/sbin/zmogrify /etc/kernel/postinst.d`
1. **REBOOT THE SYSTEM**
1. After reboot, walk through the "Getting Started" screens that the Fedora desktop presents, but ignore everything except setting up a new user. Login as the new user that you created.
1. Open a terminal and become the `root` user. Download the latest ZFS package from the "ZFS On Linux" site [https://github.com/zfsonlinux/zfs/wiki/Fedora] by running the following commands
    - `dnf install http://download.zfsonlinux.org/fedora/zfs-release$(rpm -E %dist).noarch.rpm`
    - `gpg --quiet --with-fingerprint /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux`
1. Install the ZFS packages by running the following commands (**NOTE:** The `--allowerasing` option is so that the `zfs-fuse` package can be safely removed in order to not conflict with the ZFS On Linux packages)
    - `dnf install kernel-devel`
    - `dnf install --allowerasing zfs zfs-dracut`
1. During installation the process will pause to build the "spl" and "zfs" modules. Look for a line showing the path to their locations in `/lib/modules/x.y.z/extras`. If those lines don't exist, the `kernel-devel` package you installed doesn't match your currently running kernel and you may need to restart in order to make sure they match.
1. Run the command `modprobe zfs`
   - If you get a message that zfs isn't present, the modules failed to build. Try rebooting, then re-running step 44
1. Think up a super cool name for your ZFS pool
1. Create the ZFS pool (without mounting any datasets) by running the command `zpool create -m none -O canmount=off -O atime=on -O relatime=on -O compression=lz4 -O normalization=formD -O xattr=sa -O acltype=posixacl $POOL_NAME /dev/sdx3`
1. To avoid rebuilding the "initrd" ramdisk using "dracut" each time your pool changes, run the following commands to disable the ZFS cache file and instead import pools via scan
   - `systemctl disable zfs-import-cache`
   - `systemctl enable zfs-import-scan`
   - `zpool set cachefile=none $POOL_NAME`
   - `rm /etc/zfs zpool.cache`
1. Create the OS level of ZFS pool by running the command `zfs create -p $POOL_NAME/ROOT/fedora`
1. Allow the OS level to mount by running the command `zfs set canmount=on $POOL_NAME/ROOT/fedora`
1. Re-import the pool by executing the following commands
   - `zpool export $POOL_NAME`
   - `zpool import $POOL_NAME -d /dev/disk/by-id -o altroot=/sysroot`
   - **NOTE:** It is recommended to use either "ID" or "UUID", as simple device path names can change while ID and UUID do not. This guide uses "ID" because in theory the "ID" shown is also the serial number (or similar) that you'd find on the label of the drive if you physically looked at it
1. Mount the ZFS root dataset at `/sysroot` by running the command `zfs set mountpoint=/ $POOL_NAME/ROOT/fedora`
   - **NOTE:** This works because the "altroot" was set to `/sysroot` above. `mountpoint=/` really means "mount at the root"
1. **[OPTIONAL]** Since we're using a single disk, make sure that created files write 2 copies to improve chances of data recovery (if corruption occurs) by running the command `zfs set copies=2 $POOL_NAME/ROOT/fedora`
1. **[OPTIONAL]** Create various mountpoints tuned a little better for their applications by running the following commands
   - `zfs create -o setuid=off $POOL_NAME/ROOT/fedora/home`
   - `zfs create -o mountpoint=/root $POOL_NAME/ROOT/fedora/home/root`
   - `zfs create -o setuid=off -o exec=off -o canmount=off $POOL_NAME/ROOT/fedora/var`
   - `zfs create -o com.sun:auto-snapshot=false $POOL_NAME/ROOT/fedora/var/cache`
   - `zfs create $POOL_NAME/ROOT/fedora/var/log`
   - `zfs create $POOL_NAME/ROOT/fedora/var/spool`
   - `zfs create -o com.sun:auto-snapshot=false -o exec=on $POOL_NAME/ROOT/fedora/var/tmp`
   - `zfs create $POOL_NAME/ROOT/fedora/var/lib`
1. **WARNING:** `/usr` in Fedora cannot be mounted in a separate dataset like this. Directories under `/usr` can, but not the `/usr` directory itself.
1. If you created the extra mountpoints, you must also add any of the `/var` subdirectories to bottom of `/etc/fstab` as follows

    ```
    <poolname>/ROOT/fedora/var/cache      /var/cache      zfs     defaults 0 0
    <poolname>/ROOT/fedora/var/lib        /var/lib        zfs     defaults 0 0
    <poolname>/ROOT/fedora/var/log        /var/log        zfs     defaults 0 0
    <poolname>/ROOT/fedora/var/spool      /var/spool      zfs     defaults 0 0
    <poolname>/ROOT/fedora/var/tmp        /var/tmp        zfs     defaults 0 0
    ```

1. Normally the above entries would be created automatically by ZFS when it attempted to mount them. Since we're running on `/dev/sdx4` and not in the ZFS pool, however, run the following commands to create and mount the directories manually
   - `mkdir -p /sysroot/var/cache && mount -t zfs <poolname>/ROOT/fedora/var/cache /sysroot/var/cache`
   - `mkdir /sysroot/var/lib && mount -t zfs <poolname>/ROOT/fedora/var/cache /sysroot/var/lib`
   - `mkdir /sysroot/var/log && mount -t zfs <poolname>/ROOT/fedora/var/cache /sysroot/var/log`
   - `mkdir /sysroot/var/spool && mount -t zfs <poolname>/ROOT/fedora/var/cache /sysroot/var/spool`
   - `mkdir /sysroot/var/tmp && mount -t zfs <poolname>/ROOT/fedora/var/cache /sysroot/var/tmp`
1. Update your initramfs by running the command ``dracut -fv --kver `uname -r` ``
1. Use rsync to copy all data from `/dev/sdx4` to your new ZFS pool by running the command `rsync -avxHASX / /sysroot/`
1. Run the command `ls -al /dev/disk/by-uuid` and note the UUID of the UEFI partition (hereafter `$UEFI_UUID`) and the boot partition (hereafter `$BOOT_UUID`)
1. Edit the fstab file at `/sysroot/etc/fstab`, comment out the lines beginning with `/dev/mapper/`, and insert the following two lines at the top (if they don't already exist)

    ```
    UUID=$BOOT_UUID    /boot      ext3  defaults 1 2
    UUID=$UEFI_UUID    /boot/efi  vfat  umask=0077,shortname=winnt 0 2
    ```

1. Chroot into the ZFS pool by running the command `zenter /sysroot`
1. Specific to Fedora 28, some patches to the `grub2-tools` package broke ZFS booting. Fix it by running the command `zpool status`, noting the ID of the first device in your root pool (something like `/dev/disk/by-id/wwn-0x5002538d415dc9b5-part3`) and editing the file `/etc/default/grub` to contain the following line

    ```
    GRUB_DEVICE_BOOT=/dev/disk/by-id/wwn-0x5002538d415dc9b5-part3
    ```

1. Run `lsblk -f` to get the name of the drive you're using for the ZFS pool. With that, in `/etc/default/grub`, edit the entry for `GRUB_CMDLINE_LINUX` to comment out the old line and replace it with the new information
1. Update dracut by running the command ``zmogrify `uname -r` ``
1. Exit the chroot by running the command `exit`
1. **[OPTIONAL]** If you added the extra `/var` mounts earlier, run the command `zfs unmount -a` to unmount them
1. Export the pool by running the command `zpool export $POOL_NAME`
1. **REBOOT THE SYSTEM**
1. If Fedora booted up without incident, once again become `root` in a terminal. Make sure ZFS is being used by running the command `mount | grep zfs` and looking for `$POOL_NAME/ROOT/fedora`
1. If you're now running with the ZFS pool as your root directory, congratulations! Create a backup snapshot by running the command `zfs snapshot $POOL_NAME/ROOT/fedora@firstBoot`
1. **[OPTIONAL]** Create a 4GB swap volume at `/dev/zvol/$POOL_NAME/swap` using ZFS' "ZVOL" capability by running the following command

    ```
    zfs create  <poolname>/swap \
        -V 4G \
        -o volblocksize=$(getconf PAGESIZE) \
        -o compression=zle \
        -o logbias=throughput \
        -o sync=always \
        -o primarycache=metadata \
        -o secondarycache=none \
       -o com.sun:auto-snapshot=false
    ```

1. Edit `/etc/fstab` to add the following line

    ```
    /dev/zvol/$POOL_NAME/swap none swap defaults 0 0
    ```

1. Format the swap volume by running the command `mkswap -f /dev/zvol/$POOL_NAME/swap`
1. Enable swap by running the command `swapon -av`
1. _DON'T_ enable hibernation! (Memory could be written to swap, but then swap wouldn't be accessible when the system tried to come back up)
1. Close the LUKS container on `/dev/sdx4` by running the command `cryptsetup remove /dev/sdx4`
1. Remove LUKS headers entirely from `/dev/sdx4` by running the command `dd if=/dev/urandom of=/dev/sdx4 bs=512 count=20480`
1. Verify LUKS header have been removed by running the command `cryptsetup -v isLuks /dev/sdx4` (look for "Command failed with code -1")
1. The next part is pretty hefty stuff. You will note some vital information about disk sectors, actually remove the partitions for both `/dev/sdx3` and `/dev/sdx4`, and then recreate `/dev/sdx3` to encompass all of that space. I have actually done this and it works, so hold tight and follow the instructions until the end!
1. Remove the `/dev/sdx4` partition by running the following commands
   - `parted /dev/sdx`
   - `rm 4`
   - `unit s`
   - `p`
   - (Note the starting sector of partition 3 and also the total sector size of the disk)
   - `rm 3`
   - `mkpart` (give it a name if desired, type is "bf01", starting point is starting sector of partition 3, ending point is total sector size of disk minus 1)
   - `quit`
1. Run the command `cryptsetup resize /dev/sdx3`
1. At this point, assuming no strange errors, you have your final partition layout
1. **REBOOT THE SYSTEM**
1. In a terminal, run `df -h` to check the ZFS pool's available size (it shouldn't have changed yet)
1. Resize the root ZFS partition by running the command `zpool online -e $POOL_NAME /dev/sdx3`
1. Check with `df -h` to ensure that the pool has essentially doubled in size
1. **[OPTIONAL]** ZFS can fail to mount if it thinks it's being imported on another system. In order to avoid future errors, generate a random 8 digits (4 bytes) of hexadecimal (a link for this can be found at the end of this guide) to serve as your "host id" and run the command `zgenhostid $HEX`
1. Update the initramfs by running the command ``dracut -fv --kver `uname -r` ``
1. **[OPTIONAL]** Limit ARC memory usage to half of your RAM (in bytes, must be a power of 2) by adding the below line to the file `/etc/modprobe.d/zfs.conf`

```
options zfs zfs_arc_max=8589934592   # This is roughly 8GB
```

1. Update the initramfs by running the command ``dracut -fv --kver `uname -r` ``
1. **[OPTIONAL]** ZFS stresses the use of ECC memory, as it's intended to prevent and mitigate corruption in every possible way it can. Since most systems (and this guide's) don't have ECC RAM, it's possible to set an unsupported flag that does extra checksums on buffers before writing or somesuch by editing `/etc/modprobe.d/zfs.conf` to add the following line

```
options zfs zfs_flags=0x10
```

1. Update the initramfs by running the command ``dracut -fv --kver `uname -r` ``
1. Verify by running the command `cat /sys/module/zfs/parameters/zfs_flags`

## Conclusion
That's it! You should now have a stable ZFS system on your UEFI computer that also uses LUKS encryption on all GPT partitions except `/boot` and `/boot/efi`. It's entirely possible to create new pseudo-partitions within that same LUKS encryption by utilizing ZFS datasets, as well. Nifty!


## Extra Tip - ZFS Snapshots To Test New Things:
From the csparks guide linked below comes the following suggested procedure for attempting new package installs in Fedora in a way that allows easy rollbacks:
1. `zfs snapshot $POOL_NAME/ROOT/fedora@somename`
1. (things go boom)
1. `dnf history rollback <last installed package>`
1. `zfs rollback $POOL_NAME/ROOT/fedora@somename`
1. (Try again, or run `zfs destroy $POOL_NAME/ROOT/fedora@somename` to wipe out snapshot)

## Credits
- Much of this guide (the non-LUKS parts) was based off an excellent guide found at https://www.csparks.com/BootFedoraZFS/index.md
- The idea for accomplishing a system installation without needing multiple flash drives or disks was gleaned from a guide found at https://rudd-o.com/linux-and-free-software/installing-fedora-on-top-of-zfs

## Links
- If you need to check whether your disk is Advanced Format or not, some good commands to check it can be found at https://wiki.archlinux.org/index.php/Advanced_Format
- If you need to generate some random hexadecimal, I have successfully used https://www.random.org/bytes/ to do so
