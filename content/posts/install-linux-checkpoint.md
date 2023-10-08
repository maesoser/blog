+++
title = "Enable DoH validation in Mikrotik"
date = "2023-01-19"
description = "Because starting a blog it's not difficult anymore"
draft = true

[taxonomies]
tags = ["internet", "projects"]

+++

Checkpoint L-71

- CPU: Annapurna Labs Alpine AL-21200 @1.7GHz (Dual Core ARM Cortex-A15)
- Memory: 1GB (SDRAM) DDR3
- Storage: 1GB NAND Flash, TF card (max.64GB)/MMC card
- Network: 8x Gigabit Ethernet
- USB: 1x USB 3.0 (Type A) ports


```
mkdir debian-chroot
sudo debootstrap --arch armhf --foreign bullseye debian-chroot http://deb.debian.org/debian/
# I mount an USB and copy the chroot to the USB
sudo mount /dev/sda /mnt/usb
sudo mv debian-chroot /mnt/usb
# I plug the usb into the firewall and plug the usb
ssh -oHostKeyAlgorithms=+ssh-rsa admin@192.168.1.1:/tmp/
# we fix the terminal problem
cd /mnt/usb1/debian-chroot/bin
cp bash clish
cd /mnt/usb1
# Now we give the chroot access to the sd card for format it
cp /dev/sdb* debian-chroot/dev/
umount /dev/sdb1
#  "Logs and Monitoring" -> Options -> "Eject SD card safely
# now we format it inside the chroot
chroot debian-chroot
mke2fs -t ext3 /dev/sda1
```

```
cp /mnt/usb1/debian-chroot /mnt/sd/debian-chroot
cd /mnt/sd
mount --bind /tmp /mnt/sd/debian-chroot/tmp
mount -t proc proc /mnt/sd/debian-chroot/proc
mount -t sysfs sysfs /mnt/sd/debian-chroot/sys
mount -t devpts devpts /mnt/sd/debian-chroot/dev/pts
chroot debian-chroot bash -l
```

```
touch /pfrm2.0/etc/userScript
chmod 755 /pfrm2.0/etc/userScript
echo 'mkdir -p /mnt/usb1' >> /pfrm2.0/etc/userScript
echo 'mount /dev/sda1 /mnt/usb1' >> /pfrm2.0/etc/userScript
echo 'mount -t proc proc /mnt/usb1/kali-root/proc' >>  /pfrm2.0/etc/userScript
echo 'mount -t sysfs sysfs /mnt/usb1/kali-root/sys' >> /pfrm2.0/etc/userScript
echo 'mount -t devpts devpts /mnt/usb1/kali-root/dev/pts' >> /pfrm2.0/etc/userScript
```

Change shell from cpsh to bash for admin
```
chsh -s /etc/cpshell admin
save config (OR IT WILL NOT SURVIVE NEXT REBOOT)
```


mount -t proc proc /proc
mount -t sysfs sysfs /sys
If you outside of the jail
mount -t proc proc /mnt/sd/kali-chroot/proc
mount -t sysfs sysfs /mnt/sd/kali-chroot/sys
Also be sure to add the following to the startup script. This way this gets mounted on start of the firewall (assuming you want to, which i do)

/pfrm2.0/etc/userScript
# mount sda1 because mounting happens after startup script.
mount /dev/sda1 /mnt/sd
mount -t proc proc /mnt/sd/kali-chroot/proc
mount -t sysfs sysfs /mnt/sd/kali-chroot/sys

## References:

- http://en.techinfodepot.shoutwiki.com/wiki/Check_Point_L-71W
- https://wiki.debian.org/Debootstrap
- https://www.fir3net.com/Firewalls/Check-Point/checkpoint-moving-files-using-scp.html
- https://www.spikefishsolutions.com/post/customizing-check-point-gaia-with-kali-linux
- https://www.spikefishsolutions.com/post/installing-kali-linux-on-a-checkpoint-750-smb-gaia-embedded-firewall


