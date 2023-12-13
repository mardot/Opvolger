# Ubuntu 23.10 on VisionFive 2 with external GPU
This guide could be adapted to use any gpu you have lying around, just swap out the kernel blobs and select the driver in the step
``make menuconfig``

Steps are as follows:

- update the visionfive 2 firmware to 3.8.2
- compile a custom starfive kernel (from their linux git) that includes the firmware blobs
- copy initram.fs from the starfive img (so we dont need to make it)
- flash an sd card with ubuntu 23.10 riscv preinstall using belena etcher
- delete the kernel from the ubuntu preinstall and copy your own custom kernel
- update grub

- install gnome-desktop

## Hardware

- StarFive VisionFive 2
- pcie riser card 16x to 1x (they were used for ethereum mining)
- ATI Radeon HD5450
- A 12v power supply for the GPU (old atx psu is good)
- ethernet cable to connect VisionFive2 to your router
- UART USB to TTL strictly 3.3v.

## Download

download (to `~/Downloads/SF2_2023_11_20`):
```
├── starfive-jh7110-VF2_515_v3.8.2-66-SD-minimal-desktop-wayland.img
├── u-boot-spl.bin.normal.out
└── visionfive2_fw_payload.img
```

Download the `starfive-jh7110-VF2_515_v3.8.2-66-SD-minimal-desktop-wayland.img.bz2` (202310/sd) from StarFive VisionFive 2 Support [page](https://debian.starfivetech.com/) (I used One Drive).

Download [ubuntu-23.10-preinstalled-server-riscv64+visionfive.img.xz](https://cdimage.ubuntu.com/releases/23.10/release/ubuntu-23.10-preinstalled-server-riscv64+visionfive2.img.xz) from https://ubuntu.com/download/risc-v
Unzip it to `ubuntu-23.10-preinstalled-server-riscv64+visionfive2.img`.

```bash
$ mkdir -p ~/Downloads/SF2_2023_11_20
$ cd ~/Downloads/SF2_2023_11_20
$ wget https://github.com/starfive-tech/VisionFive2/releases/download/VF2_v3.8.2/visionfive2_fw_payload.img
$ wget https://github.com/starfive-tech/VisionFive2/releases/download/VF2_v3.8.2/u-boot-spl.bin.normal.out
$ bzip2 -d ~/Downloads/SF2_2023_11_20/starfive-jh7110-VF2_515_v3.8.2-66-SD-minimal-desktop-wayland.img.bz2
```

## Update the VisionFive 2 firmware and uboot to v3.8.2

I used the docker image to tftp the file to my visionfive [image](https://hub.docker.com/r/pghalliday/tftp)

```bash
$ cd ~/Downloads/SF2_2023_11_20
# startup the docker tftp server and expose the current directory to /var/tftpboot
# The first time you will see this message, but the second time you will see nothing
$ docker run -p 0.0.0.0:69:69/udp -v $(pwd):./ -i -t pghalliday/tftp
Unable to find image 'pghalliday/tftp:latest' locally
latest: Pulling from pghalliday/tftp
fae91920dcd4: Pull complete 
0bb6771b2292: Pull complete 
Digest: sha256:59f843a93d62ac6e2fa475ee3d1a3786464fe5ea0334e6f1ea79bdec837f09fa
Status: Downloaded newer image for pghalliday/tftp:latest
```

Do NOT hit Ctrl+C! We need this server until we are done flashing!

Connect your VisionFive 2 via ethernet cable to your router.
Boot up your VisionFive 2 without a SD-card and the USB to TTL connected (see [link](https://doc-en.rvspace.org/VisionFive2/PDF/VisionFive2_QSG.pdf) for more information)

My machine (with docker) has ip-address 192.168.2.29 and I give my VisionFive 2 the ip 192.168.2.222

This will be the commands (for the Ubuntu firmware):

```bash
# set the ip of the VisionFive 2, and of the server (where docker is running)
$ setenv ipaddr 192.168.2.222; setenv serverip 192.168.2.29
# check connection
$ ping 192.168.2.29
# Initialize SPI Flash
$ sf probe
# Download and Update SPL binary
$ tftpboot 0xa0000000 ${serverip}:u-boot-spl.bin.normal.out
$ sf update 0xa0000000 0x0 $filesize
# Download and Update U-Boot binary
$ tftpboot 0xa0000000 ${serverip}:visionfive2_fw_payload.img
$ sf update 0xa0000000 0x100000 $filesize
# Reset the default (default load options)
$ env default -f -a
# Save your changes
$ env save
```

You can turn off the VisionFive 2 again and hit Ctrl+C on the other machine to stop docker.

## Create the SD-card

I used `balenaEtcher` to flash a SD-card with `ubuntu-23.10-preinstalled-server-riscv64+visionfive2.img`.

## Building the Linux Kernel

Building a kernel of the source code from StarFive
Compile kernel on desktop or laptop. Takes me 5 minutes using a AMD Ryzen 5 5600g APU.
Compling the kernel on the VisionFive 2 takes around 1.5 hrs give or take.

```bash
$ mkdir ~/Downloads/SF2_2023_11_20/visionfive2
$ cd ~/Downloads/SF2_2023_11_20/visionfive2
# we need the firmware for AMD / ATI cards, we do not need history so depth 1
$ git clone --depth 1 git://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git linux-firmware

# Check out the last kernel, we do not need history so depth 1
$ git clone --branch VF2_v3.8.2 --depth 1 https://github.com/starfive-tech/linux.git

# Obtain the ESWIN 6600u wifi driver from starfives buildroot repo
$ git clone --branch VF2_v3.8.2 https://github.com/starfive-tech/buildroot.git
$ sudo cp $VF2_WORK_DIR/buildroot/package/starfive/usb_wifi/ECR6600U_transport.bin /$VF2_WORK_DIR/visionfive2/linux-firmware

```
reference https://github.com/starfive-tech/linux/tree/JH7110_VisionFive2_upstream

```bash
# go to checkout code
$ cd ~/Downloads/SF2_2023_11_20/visionfive2/linux

# create the default starfive .config with all you need for only the StarFive VisionFive 2
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- starfive_jh7110_defconfig # starfive_jh7110_defconfig -> we need PCI-e which is not enabled in starfive_visionfive2_defconfig

```
### Customize the kernel with menuconfig
```
# open the menu
make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv menuconfig
```


Direct the firmware loader to ../linux/firmware where the firmware blobs are situated for the radeon drivers
```
Device Drivers ---> [HIT ENTER]
  Generic Driver Options ---> [HIT ENTER]
    Firmware loader ---> [HIT ENTER]
      () Build named firmware blobs into the kernel binary [HIT ENTER]
      (/lib/firmware) Firmware blobs root directory [change it to ../linux-firmware]
```

enter this in the line:

(the bin files for ATI Radeon 5450 and the ESWIN 6600U usb wifi)

```ini
radeon/CYPRESS_uvd.bin radeon/CEDAR_smc.bin radeon/CEDAR_me.bin radeon/CEDAR_pfp.bin radeon/CEDAR_rlc.bin ECR6600U_transport.bin
```

Select Exit,Exit

We need some stuff for snapd:

```
Device Drivers -> 
    Block devices -> [HIT ENTER]
      <*> RAM block device support [HIT SPACE 2x]
```

Select Exit

```
Device Drivers -> 
    Network device support ---> [HIT ENTER]
      Wireless LAN ---> [HIT ENTER]
        [*] USB WIFI ECR6600U hit space
```

Select Exit,Exit,Exit

```
Device Drivers --->
  Graphics support ---> [HIT ENTER]
    <*> ATI GPU [HIT SPACE]
    <*> AMD GPU [HIT SPACE 2x]         
    [*] Enable amdgpu support for SI parts [HIT SPACE]
    [*] Enable amdgpu support for CIK parts [HIT SPACE]
    ACP (Audio CoProcessor) Configuration ---> [HIT ENTER]
      [*] Enable AMD Audio CoProcessor IP support [HIT SPACE]
```

Select Exit,Exit

```
Device Drivers --->
  Sound card support ---> [HIT ENTER]
    Advanced Linux Sound Architecture ---> [HIT ENTER]
      HD-Audio ---> [HIT ENTER]
        <*> HD Audio PCI [HIT SPACE 2x]
        <*> Build HDMI/DisplayPort HD-audio codec support [HIT SPACE 2x]
```

Select Exit,Exit,Exit,Exit

We need more some stuff for snapd:

```
File systems  -> [HIT ENTER]
  Miscellaneous filesystems -> [HIT ENTER]
    <*> SquashFS 4.0 - Squased file system support [HIT SPACE 2x]
    [*] Squashfs XATTR support [HIT SPACE]
    [*] Include support for ZLIB compressed file systems
    [*] Include support for LZ4 compressed file systems [HIT SPACE]
    [*] Include support for LZO compressed file systems [HIT SPACE]
    [*] Include support for XZ compressed file systems [HIT SPACE]
    [*] Include support for ZSTD compressed file systems [HIT SPACE]
```

Select Exit,Exit

We need CONFIG_SECCOMP and CONFIG_SECCOMP_FILTER for the `epiphany` browser

```
General architecture-dependent options  ---> [HIT ENTER]
  [*] Enable seccomp to safely execute untrusted bytecode -> [HIT SPACE]
```

Select Exit,Exit

Yes You wish to save your new configuration!

Delete the previous Ubuntu kernels
```
sudo rm vmlinuz* System.map-* initrd.img* config-* dtb*
```

```bash
# Compile the kernel
# make custom kernel in directory
# make modules

$ make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j 6

$ make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- INSTALL_PATH=~/Downloads/visionfive2/customkernel zinstall

#IF that went well push straight to the SD cards boot directory

$ sudo make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- INSTALL_PATH=/media/$USER/cloudimg-rootfs/boot zinstall

$ make modules_install INSTALL_MOD_PATH=~/Downloads/SF2_2023_11_20/visionfive2/customkernel/

IF that went well push straight to the sd card
$ sudo make modules_install INSTALL_MOD_PATH=/media/$USER/cloudimg-rootfs/

```

The kernel is Done! Now copy it to the boot partition on the SD-card!

## BOOT/ROOT Copy kernel

We need the initrd-img of 5.15.0 from VisionFive 2, so we don't have to make it ourselves.

```bash
# create a loop device of image
sudo losetup -f -P ~/Downloads/SF2_2023_11_20/starfive-jh7110-VF2_515_v3.8.2-66-SD-minimal-desktop-wayland.img
# find your loop device
$ losetup -a # or -l
/dev/loop0: []: (/home/$USER/Downloads/SF2_2023_11_20/starfive-jh7110-VF2_515_v3.8.2-66-SD-minimal-desktop-wayland.img)
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0   3,9G  0 loop 
├─loop0p1   259:7    0     2M  0 part 
├─loop0p2   259:8    0     4M  0 part 
├─loop0p3   259:9    0   100M  0 part 
└─loop0p4   259:10   0   3,8G  0 part 
....
....
....
....
# create dir
$ mkdir -p ~/Downloads/SF2_2023_11_20/visionfive2/mount
# mount to dir
$ sudo mount /dev/loop0p3 ~/Downloads/SF2_2023_11_20/visionfive2/mount
# copy initrd img
$ cp ~/Downloads/SF2_2023_11_20/visionfive2/mount/initrd.img-5.15.0-starfive ~/Downloads/SF2_2023_11_20/visionfive2
# umount
$ sudo umount /dev/loop0p3
# detach loop device
$ sudo losetup -d /dev/loop0
```

We have the SD-card inserted (again). Mount the sd-card (cloudimg-rootfs)
Now we need to mount the first partition of the SD-card. my is mounted on `/media/$USER/cloudimg-rootfs/` 

```bash
previously we copied to our sd /boot directory
├── config-5.15.0-starfive
├── System.map-5.15.0-starfive
├── vmlinuz-5.15.0-starfive

now we need
├── jh7110-visionfive-v2.dtb
└── initrd.img-5.15.0-starfive

# copy the dtb (Device Tree) to the sd-card
$ sudo cp arch/riscv/boot/dts/starfive/jh7110-visionfive-v2.dtb /media/$USER/cloudimg-rootfs/boot/
# copy initrd to the sd-card
$ sudo cp ~/Downloads/SF2_2023_11_20/visionfive2/initrd.img-5.15.0-starfive /media/$USER/cloudimg-rootfs/boot/

# we have now all the 5 files we need
```

### Grub custom menu entry

```bash
$ sudo nano /media/$USER/cloudimg-rootfs/etc/grub.d/40_custom
# Copy the text above
```

```ini
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'Ubuntu StarFive VisionFive 2 5.15.0-starfive' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-5-15-0-starfive' {
        load_video
        insmod gzio
        if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root 40249bdb-adbf-44bf-a50e-74b81c1200d7
        echo    'Loading Linux 5.15.0-starfive ...'
        linux   /boot/vmlinuz-5.15.0-starfive root=LABEL=cloudimg-rootfs ro  efi=debug earlycon sysctl.kernel.watchdog_thresh=60
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-5.15.0-starfive
        echo    'Loading device tree blob...'
        devicetree      /boot/jh7110-visionfive-v2.dtb
}
```

```bash
$ sudo nano /run/media/$USER/cloudimg-rootfs/etc/grub.d/40_custom
# Copy the text above
```
## Edit Uboot boot options
Put the Sd card in and boot the vf2 with the gpu connected

The boot options will be set to mmc device 1 partiton 3, we need to change that to partition 1

```
StarFive # edit /boot/extlinux/extlinux.conf
edit: emmc /dev/mmcblk1p1h
StarFive # saveenv
Saving Environment to SPIFlash... Erasing SPI flash...Writing to SPI flash...done
OK

StarFive # env edit bootcmd_mmc1
edit: setenv devnum 1; run mmc_boot
StarFive # env edit bootcmd 
edit: run distro_bootcmd                   
StarFive # saveenv 
Saving Environment to SPIFlash... Erasing SPI flash...Writing to SPI flash...done
OK
StarFive # reset   
resetting ...
```

#todo add usb boot options

## Update Grub

Put the SD-Card in the VisionFive 2 with an external GPU and use the USB to TTL.

Turn on the VisionFive 2

If booting is not working run the command again

```
setenv default -f -a
setenv save
reset
```


You will get to setup your ubuntu password (default is ubuntu/ubuntu).

```bash

Ubuntu 23.10 ubuntu ttyS0

ubuntu login: ubuntu
Password: 
You are required to change your password immediately (administrator enforced).
Changing password for ubuntu.
Current password: 
New password: 
Retype new password: 
Welcome to Ubuntu 23.10
....

```

Now we will change the default boot option of Grub
We will edit `etc/default/grub`

The default boot option is now `GRUB_DEFAULT=0` we will change that to `GRUB_DEFAULT=2`. The boot options start counting with 0, so we need option number 3 (That is 2).

```bash
ubuntu@ubuntu:~$ sudo nano /etc/default/grub
# now change `GRUB_DEFAULT=0` to `GRUB_DEFAULT=2`
# We will now update grub with the custom kernel and menuentry
ubuntu@ubuntu:~$ sudo update-grub

# turn it off
ubuntu@ubuntu:~$ sudo halt
```


## Boot Visionfive 2 with external GPU

We will start the Visionfive 2 with the external GPU.

We will use the USB to TTL to see that every thing is working!

```bash
$ minicom

# After boot up, login with ubuntu and your password

Ignore updating kernel headers

# First update all the packages!
$ sudo apt update
$ sudo apt upgrade
# after install updates ignore kernel warning! Restart some services


# install Gnome desktop gui
ubuntu@ubuntu:~$ sudo apt install gnome-desktop

ubuntu@ubuntu:~$ sudo reboot
```

Now Gnome-desktop will startup after booting.

#todo
write how to swap kernel headers for apt to work properly

## Network

Fix network manager in Gnome (so you can control network in Gnome)

Need reboot to get it working!

## Screenshot!


ATI Radeon 5450:
add the screencap





