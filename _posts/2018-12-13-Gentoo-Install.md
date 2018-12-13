---
layout:		post
title: 		Gentoo Install
subtitle: 	gentoo install
date:		2018-12-13
author:		Tim
header-img: 	img/post-bg-2015.jpg
catalog: true
tags:
    - gentoo

---

### wifi

```
> vi /etc/wpa_supplicant/wpa_supplicant.conf
network={
    ssid="TP-LINK"
    psk="123456"
}
> wpa_supplicant -B -Dnl80211 -iwlp0s11u1 -c /etc/wpa_supplicant/wpa_supplicant.conf
```

### disk

```
> parted -a optimal /dev/sda
> mklabel msdos
> mkpart primary ext2 1 513
> set 1 boot on
> mkpart primary ext4 513 -1
> set 2 lvm on
> mkfs.ext2 /dev/sda1
> mkfs.ext2 -L "boot" /dev/sda1
> cryptsetup -y -y -c aes-xts-plain64 -s 512 -h sha512 -i 5000 --use-random luksFormat /dev/sda2
        > YES
> password
> cryptsetup luksDump /dev/sda2
> cryptsetup luksOpen /dev/sda2 gentoobox
```

### lvm

```
> pvcreate /dev/mapper/gentoobox
> pvdisplay
> vgcreate gentoo /dev/mapper/gentoobox
> vgdisplay
> lvcreate -C y -L 4G gentoo -n swap
> lvcreate -l +100%FREE gentoo -n root
> lvdisplay
> vgscan
> vgchange -ay
> mkswap /dev/mapper/gentoo-swap
> swapon /dev/mapper/gentoo-swap
> mkfs.ext4 /dev/mapper/gentoo-root
```

### mount

```
> mount /dev/mapper/gentoo-root /mnt/gentoo/
> mkdir /mnt/gentoo/boot
> mount /dev/sda1 /mnt/gentoo/boot
> lsblk
```


### download stage

```
> tar xvjpf stage3-amd64-20171208**.tar.bz2
```

### config

```
> vi /mnt/gentoo/etc/portage/make.conf

CFLAGS="-march=native -02 -pipe"
CXXFLAGS="${CFLAGS}"
CHOST="x86_64-pc-linux-gnu"
MAKEOPTS="-j5"
USE="cryptsetup crypt plymouth pulseaudio icu python branding png jpeg jpg bindist"
VIDEO_CARDS=""
ALSA_CARDS=""
CPU_FLAGS_X86="mmx sse sse2"

> cp -L /etc/resolv.cof /mnt/gentoo/etc/
> mkdir /mnt/gentoo/etc/portage/repos.conf
> cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
> mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
```

### mount

```
> mount -t proc /proc /mnt/gentoo/proc
> mount --rbind /sys /mnt/gentoo/sys
> mount --rbind /dev /mnt/gentoo/dev
> mount --make-rslave /mnt/gentoo/sys
> mount --make-rslave /mnt/gentoo/dev
> mkdir /mnt/gentoo/hostrun
> mount --bind /run /mnt/gentoo/hostrun/
> chroot /mnt/gentoo /bin/bash
> source /etc/profile
> mkdir /run/lvm
> mount --bind /hostrun/ /run/lvm
> ls /run/
```

### update
```
> emerge-webrsync
> emerge --sync
> eselect profile list
> emerge -av vim
> emerge -av gentoo-sources genkernel-next plymouth lvm2
> cd /etc/portage/package.use
> mv ._cfg000_iputils genkernel-next
> touch zz-template
> vim genkernel-next
#net-misc/iptutils -caps -filecaps
> emerge -av gentoo-sources genkernel-next plymouth lvm2
> emerge -uvDNa @world
```

### fstab

```
> vim /etc/fstab

/dev/sda1           /boot   ext2    noatime             1 2
/dev/gentoo/root    /       ext4    defaults,relatime   0 1
/dev/gentoo/swap    none    swap    sw                  0 0
```

### make genkernel

```
>genkernel --makeopts=-j5 --menuconfig --lvm --luks all
    > General setup
        > Local version -append to kernel release =>-gentoo
    > Device Drivers
        > Multiple devices driver superport (RAID and LVM)
            > Device mapper support => *
            > Crypt target support => *
            > Snapshot target =>*
            > Mirror target =>*
            > Multipath target =>*
            > I/O Path Selector based on the number of in-flight I/Os =>*
            > I/O Path Selector based on the service time =>*
        > HID support
            >Special HID drivers
                >HID Multitouch panels => M
            
    > Cryptogra[hic API
        > SHA1 digest algorithm =>-*- (all)
        > Fixed time AES clipher =>*
        > AES cipher algorithms (AES-NI) =>*
        > Serpent cipher algorithm =>*
        > Twofish cipher algorithm (x86_64) =>*
        > User-space interface for ... =>* (all)
```

### grub

```
> echo "sys-boot/grub mount device-mapper" > /etc/portage/package.use/grub
> memrge -av grub
> vim /etc/default/grub
GRUB_PRELOAD_MODULES=lvm
GRUB_ENABLE_CRYPTODISK=y
GRUB_DEVICE=/dev/ram0
GRUB_CMDLINE_LINUX="crypt_root=/dev/sda2 real_root=/dev/mapper/gentoo-root rootfstype=ext4 dolvm quiet splash"
> grub-install --modules="linux crypto search_fs_uuid luks lvm" --recheck /dev/sda
> grub-mkconfig -o /boot/grub/grub.cfg
```

### User

```
> passwd
> exit
> mount --bind /run /mnt/gentoo/hostrun
> chroot /mnt/gentoo /bin/bash
> mount --bind /hostrun/lvm /run/lvm/
> grub-mkconfig -o /boot/grub/grub.cfg
> useradd -m -G users,wheel,audio,video -s /bin/bash susan
> passwd susan
> rm -i stage3-***.tar.bz2
> vim /etc/hosts
127.0.0.1   gentoobox localhost
::1         gentoobox
```

### server

```
> emerge -av cronie syslog-ng dhcpcd
> rc-update add cronie default
> rc-update add syslog-ng default
> rc-update add sshd default
> rc-update add lvm default
> rc-update add dhcpcd default
> emerge -av wireless-tools net-tools net-wireless/wpa_supplicant
> plymouth-set-default-theme --list
> plymouth-set-default-theme solar
> cat /etc/plymouth/plymouthd.conf
> vim /etc/genkernel.conf
?PLY
PLYMOUTH="yes"
PLYMOUTH_THEME="solar"
```

### umount

```
> genkernel --lvm --luks initramfs
> grub-mkconfig -o /boot/grub/grub.cfg
> umount /run/lvm/
> umount /hostrun/
> exit
> umount -l /mnt/gentoo/dev{/shm,/pts,}
> umount -R /mnt/gentoo
> reboot
```
