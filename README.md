# zfs-root-from-ubuntu-xubuntu-lubuntu-.iso
How to boot ubuntu variants from a zfs filesystem using only a 'live cd' 16.04 / xenial ubuntu environment.

This is written presuming lubuntu. Use xubuntu or the variant of your choice where you see lubuntu below.
This is written presuming the installation will be on /dev/sda .  Replace /dev/sda with  the /dev/ path to the drive(s) that will hold the initial zfs file system and be used as boot targets.

1) Boot into 'try lubuntu' live-cd environment. Open a terminal, sudo apt-get install zfs-initramfs ssh
(ssh is optional, it's often easier to 'cut and paste' from an ssh session).

2) Use the app gparted to partition sda, leave 512mb before first partition.   Set it with the 'boot' flag, type 'unformatted'.  Do the same to all other drives that will be part of the zpool.  It is fine to install to just one drive, then create a mirror later.

3) zpool create -o ashift=12 -O compression=lz4 zfsroots /dev/sda1
The above command if for an install to a single drive.  The command is more complex if creating mirrors, raidz setups, etc. Consult the zpool command to learn how to specify more complex installs.  'zfsroots' is arbitrary.

4) zfs create -V 30G zfsroots/s2
's2' is arbitrary, you could use 'lubuntu-image' or similar.

5) Give 'grub' a target it understands in /dev.
cat >  /etc/udev/rules.d/90-zfs.rules 
KERNEL=="sd*[!0-9]", IMPORT{parent}=="ID_*", SYMLINK+="$env{ID_BUS}-$env{ID_SERIAL}"
KERNEL=="sd*[0-9]", IMPORT{parent}=="ID_*", SYMLINK+="$env{ID_BUS}-$env{ID_SERIAL}-part%n"
^D

6) udevadm trigger

7) Use the provided 'install lubuntu' icon to install lubuntu to s2.  Choose  "something else" for the install target.  Create a new partition table on 'z0' which should appear on the list of drives. Create a new partition single partition for all of z0, set it to be the root.  I chose format ext4, no swap.  Continue with the install, ignore the 'no swap' complaint.

8) At the end of the install, do 'continue without boot loader'.  Ignore the complaint.
9) At the end of the install cleanup, choose 'continue testing'.

10) 
zpool export zfsroots
zpool import -d /dev/disk/by-id/ zfsroots

11) 
zfs create zfsroots/std
mkdir /mntlcl
mount /dev/zvol/zfsroots/s2-part1 /mntlcl
rsync -avPX /mntlcl/. /zfsroots/std/.

12)
comment everything out in /zfsroots/std/etc/fstab

in /zfsroots/std/etc/default/grub change to:
GRUB_CMDLINE_LINUX="boot=zfs rpool=zfsroot bootfs=zfsroot/std"

cp /etc/udev/rules.d/90* /zfsroots/std/etc/udev/rules.d/
cp /etc/resolv.conf /zfsroots/std/etc/

13)
for d in proc sys dev;do mount --bind /$d /zfsroots/std/$d;done
chroot /zfsroots/std
apt-get install zfs-initramfs
update-grub
grep -i 'ROOT=ZFS' /boot/grub/grub.cfg
#confirm somthing like linux	/std@/boot/vmlinuz-4.4.0-21-generic root=ZFS=zfsroots/std ro boot=zfs rpool=zfsroot bootfs=zfsroot/std

exit
for d in proc sys dev;do umount /zfsroots/std/$d;done

grub-install  --boot-directory=/zfsroots/std/boot /dev/sda
#(if the zpool create command involved multiple drives, repeat this for all the drives in the command).

zfs snapshot zfsroots/std@pristine

That's it.

Reboot, remove the install media and/or change bios to use sda. 
There should be no complications, upon reboot you should be up and running on your zfs root partition.

Best to all,

Harry Coin
