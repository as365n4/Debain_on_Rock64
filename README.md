# Debain on Rock64

#### 1.)  Download the firmware file and partition image from Debians Server

https://d-i.debian.org/daily-images/arm64/daily/netboot/SD-card-images/firmware.rock64-rk3328.img.gz

https://d-i.debian.org/daily-images/arm64/daily/netboot/SD-card-images/partition.img.gz

#### 2.)  Merge the firmware file and partition image into bootable Debian image

`zcat firmware.rock64-rk3328.img.gz partition.img.gz > rk64_debian.img`

#### 3.)  Flash bootable Debian image onto SD-Card

`lsblk` find device name of your SD-Card

`sudo dd if=/dev/zero of=/dev/sdX bs=446 count=32770` wipe the Boot Sector of your SD-Card

`sudo dd if=rk64_debian.img of=/dev/sdX bs=4M conv=fsync` flash Debian image to your SD-Card

replace X with the device letter of your SD-Card

#### 4.)  Download the U-Boot package from Debians Server

https://packages.debian.org/bookworm/arm64/u-boot-rockchip/download

extract the idbloader.img and u-boot.itb files from the package,
both files are located in u-boot-rockchip_X_arm64.deb/data.tar.xz/./usr/lib/u-boot/rock64-rk3328/

#### 5.)  Prepare the eMMC-Module for the Rock64 SBC (16GB eMMC module (30310400 sectors))

`sudo fdisk /dev/sdX`

type `o` this will clear out any partitions on the drive
, type `p` to list partitions, there should be no partitions left
, type `n` for new partition, then `p` for primary, `1` for the first partition on the drive
, `32768` for the first sector, and `1056767` for the last sector, then type `a`
, then type `n` for new partition, then `p` for primary, `2` for the second partition on the drive
, `1056768` for the first sector, and `28213246` for the last sector, then type `n`
, then `p` for primary, `3` for the third partition on the drive, `28213247` for the first sector
, and `30310399` for the last sector, then type `t`, and `3` for the third partition, and `82` for the Hex Code
, then write the partition table and exit by typing `w`

(steps above create 500M for `/boot`, 12.9GB for `/` and 1GB for `swap`)

We will format the newly created partitions later with the Debian Installer.

The 500M partition format as ext2 and set mount point to /boot.
The 12.9GB partition format as ext4 and set mount point to /.
The 1 GB partition format as swap.

#### 6.)  Flash U-Boot (Bootloader) onto the eMMC-Module for the Rock64 SBC

`lsblk` find device name of your eMMC-Module

`sudo dd if=idbloader.img of=/dev/sdX seek=64 conv=notrunc` replace X with the device letter of your eMMC-Module

`sudo dd if=u-boot.itb of=/dev/sdX seek=16384 conv=notrunc`

once finished, unmount the eMMC-Module

#### 7.)  Install the eMMC-Module onto your Pine64 Rock64 SBC, insert the SD-Card, connect HDMI, Mouse and Keyboard, USB to serial (UART) adapter, power it up and follow the Debian Installer.

`sudo screen /dev/ttyUSB0 1500000`  connects you to the serial output of the Rock64 `CTRL a k` exits screen

    Pin 6 –> Ground
    Pin 8 –> TX
    Pin 10 –> RX

The Debain Installer will fail at “Making the System bootable”, ignore this and continue with the next step to finish the installation. (bootloader is installed already)

#### 8.)  Once installation is finished, add Firmware for Rockchip CDN DisplayPort Controller

`nano /etc/apt/sources.list`  amend as below

    # deb http://deb.debian.org/debian bookworm main
    
    deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
    deb-src http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
    
    deb http://deb.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
    deb-src http://deb.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
    
    #deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
    #deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
  
`apt update`
  
`apt install firmware-misc-nonfree` contains –> rockchip/dptx.bin
  
#### 9.)  Perform system update, enable filesystem check at boot and enable sudo
  
`apt update`

`apt upgrade`

`apt full-upgrade`

`apt autoremove`

`apt autoclean`

use `lsblk` to find correct block device for tune2fs

`tune2fs -c 1 /dev/mmcblkXp1`

`tune2fs -c 1 /dev/mmcblkXp2`
  
`apt install sudo`
 
`adduser youruser sudo`
 
#### 10.)  Hide kernel messages during boot
 
`nano /etc/sysctl.conf`  amend as below

    # Uncomment the following to stop low-level messages on console
    kernel.printk = 3 4 1 3
 
#### 11.)  If you operate more than one Rock64 on the same network, you may need MAC address spoofing
 
`ip link show end0`  shows current MAC address

If the same MAC address is present multiple times on the same network, then do steps below or the network will not work !

`sudo nano /etc/network/interfaces` amend as below

    # This file describes the network interfaces available on your system
    # and how to activate them. For more information, see interfaces(5).
    
    source /etc/network/interfaces.d/*
    
    # The loopback network interface
    auto lo
    iface lo inet loopback
    
    # The primary network interface
    allow-hotplug end0
    iface end0 inet dhcp
        hwaddress ether da:74:87:XX:XX:XX

change the last 3 bits to your liking, DO NOT change the first 3 bits (reserved for Manufacturer)

`reboot`  once board is up, check with `ip link show end0` for success

#### 12.) Install U-Boot onto your Rock64 to keep U-Boot up-to-date

`apt install u-boot-rockchip u-boot-menu` (flash-kernel is installed already)

`u-boot-update`

`nano /boot/extlinux/extlinux.conf` check boot menu config

`lsblk` find correct block device for U-Boot

`u-boot-install-rockchip /dev/mmcblkX`  flash U-Boot onto block device, replace X with the device number of your eMMC-Module

`reboot`

#### Done, enjoy your setup.
