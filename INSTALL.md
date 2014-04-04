#Arch Linux install

##Target system:

###Hardware

    Lenovo ThinkPad Edge E325
    4GB 1333MHz DDR3 RAM by default, upgraded to motherboard max of 8GB
    320GB 7200RPM HDD by default, upgraded to 120GB Samsung 840 EVO
    AMD E450 APU, 1.65GHz dual-core

###Software

    MBR, not GPT, because GPT is change, and I'm afraid of change
    Unencrypted /boot, ext4, with GRUB
    Rest of drive to DM-Crypt (LUKS)
    btrfs root filesystem, everything in one (no separate /home)
    No swap - 8GB RAM + SSD - Don't need swap, complicates partitioning (need LVM)
    No UEFI - BIOS legacy boot, because UEFI is a pain in the ass to deal with in 2014


#Install
##Boot from installation media
###Set keyboard layout, font, locale

    loadkeys fi
    setfont Lat2-Terminus16

Uncomment en_US.UTF-8, fi_FI.UTF-8 so we can use those locales

    nano /etc/locale.gen
    locale-gen
    export LANG=en_US.UTF-8

###Setup networking for install
We're double-paranoid so we're going to both connect wirelessly, and enable DHCP for Ethernet.

    wifi-menu
    systemctl enable dhcpcd.service

###Set up partitions and mount them
Two partitions, start default + end +2G, and start default + end default, and set first partition as bootable

    fdisk /dev/sda
    mkfs.ext4 /dev/sda1
    cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/sda2
    crypsetup open /dev/sda2 root
    mkfs.btrfs /dev/mapper/root
    mount /dev/mapper/root /mnt
    mkdir /mnt/boot
    mount /dev/sda1 /mnt/boot

###Select mirror

Remove mirrors from Five Eyes (AUS+Canada+NZ+UK+US) and Russia, and move Finnish mirror to top of list.

    nano /etc/pacman.d/mirrorlist

###Install base system

    pacstrap -i /mnt base

###Generate fstab

    genfstab -U -p /mnt >> /mnt/etc/fstab

Make sure / is being mounted with ssd,discard options

    nano /mnt/etc/fstab

###Chroot and base system config

    arch-chroot /mnt /bin/bash

###Set keyboard layout, font, locale, timezone

Uncomment en_US.UTF-8, fi_FI.UTF-8

    nano /etc/locale.gen
    locale-gen
    echo LANG=en_US.UTF-8 > /etc/locale.conf
    export LANG=en_US.UTF-8
    nano /etc/vconsole.conf
        KEYMAP=fi
        FONT=Lat2-Terminus16
    ln -s /usr/share/zoneinfo/Europe/Helsinki /etc/localtime
    hwclock --systohc --utc

###Set hostname

    echo Iris > /etc/hostname
    nano /etc/hosts

hosts file:

    #
    # /etc/hosts: static lookup table for host names
    #

    #<ip-address>   <hostname.domain.org>   <hostname>
    127.0.0.1   localhost.localdomain   localhost   Iris
    ::1     localhost.localdomain   localhost

    # End of file


###Enable DHCP service
This will give us DHCP automatically when we reboot into the installed base system

    systemctl enable dhcpcd.service

###Install packages for wireless

    pacman -S dialog wpa_supplicant wpa_actiond

###Edit ramdisk config, and generate a new one

Add "encrypt" before "filesystems" in HOOKS, if using LVM you also need to add "lvm2" 

    nano /etc/mkinitcpio.conf
    mkinitcpio -p linux

###Set root password

    passwd

###Install and setup GRUB

    pacman -S grub
    grub-install --target=i386-pc --recheck /dev/sda

Add to GRUB_CMDLINE_LINUX_DEFAULT the following: cryptdevice=/dev/sda2:root

    nano /etc/default/grub
    grub-mkconfig -o /boot/grub/grub.cfg

###Exit chroot, unmount partitions, close LUKS, reboot

    exit
    umount -R /mnt
    cryptsetup close root
    reboot
