title: Arch Setup
date: 2018-01-10T16:27:09Z
status: Draft
tags: Linux, Arch


# Notes on Arch Linux Setup

Setup of Arch Linux system for the CUEH course.
This is done in VMware

## SEtup

New Image,
Linux 4.X kernel

## Initial Install

SEtup keyboard

~~~
# loadkeys uk
~~~

Time to use NTP

~~~
# timedatectl set-ntp true
# timedatectl status
~~~

## Disk Partitioning

We are going to be stuck with MBR mode (although a 2TB disk wouldnt be appropriate for this image anyway)
We do have some options here, and could seperate out /home etc. 

 - 200M bootable partition
 - Rest for data partition

Work out the partition we need to change
~~~
fdisk -l
~~~

Make use of the Snazzy graphical partition manager.

~~~
cfdisk /dev/sda
~~~

Create the boot partition

 - type DOS
 - size 200M
 - Bootable

Then the Other

 - Size <whatever>

Finalise by selecting write then quit.

Format to ext4

~~~
mkfs.ext4 /dev/sda1
mkfs.ext4 /dev/sda2
~~~

And Mount

~~~
mount /dev/sda2/ /mnt
mkdir /mnt/boot
mount /dev/sda1/ /mnt/boot
~~~

## Install Base Packages

NOTE: We may want to fuck about with mirrors here

Install the base packages

~~~
pacstrap /mnt base
~~~

Generate the Fstab 

~~
genfstab -U /mnt >> /etc/etc/fstab

## Chroot and start installing things


~~~
arch-chroot /mnt
~~~

### Localization

Normally the time is UTC (/etc/localtime)
We can change this to UK time

~~~
# ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
~~~

Change the Language

Edit /etc/locale.gen

Uncomment en_GB.UTF-8  (Ctrl-w in nano)

And run the command to generate
~~~
#locale-gen
...
#echo LANG=en_GB.UTF-8 > /etc/locale.conf
~~~

Also hardcode the keyboard

~~~
echo KEYMAP=uk >> /etc/vconsole.conf
~~~

### Networking

~~~
echo "marblearch" >> /etc/hostname
~~~

And the /etc/hosts file

~~~
127.0.0.1	localhost
::1		localhost
127.0.1.1	MarbleArch
~~~

### Users

Set root password

~~~
passwd
~~~

Create "Standard" user

~~~
useradd -m -G wheel,users -s /bin/bash user
passwd user
~~~

Add to Sudoers

~~~
pacman -S Sudo
~~~

Edit /etc/sudoers  Under User Privilege specification

## Install Bootloaders

~~~
pacman -S grub os-prober
~~~

And install (Check the drive)

~~~
grub-install --recheck --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
~~~

## Restart

Exit chroot
~~~
exit
~~~
unmount 

~~~
umount -R /mnt
~~~

Remove USB in Options

Resart

~~~
reboot
~~~



# Notes for Med
http://grbd.github.io/posts/2016/11/06/using-the-mbed-cli/
