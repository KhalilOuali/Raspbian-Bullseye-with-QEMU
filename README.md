# Emulating Raspbian Bullseye with QEMU (arm64, raspi3b) on Ubuntu 22.04 LTS

This guide is a summary of the steps I had to take to emulate Raspbian Bullseye using QEMU, after a lot of online research.

> Last tried Jan 2023 on a Lubuntu 22.04 LTS VM

## Check the releases page for a ready archive that might save you the hassle.

## Prerequisites
* (Recommended) Update your system <br>
$ `sudo apt update && sudo apt upgrade`

* Install qemu packages <br>
$ `sudo apt install qemu-system-arm qemu-utils`

## Disclaimers
* Check your qemu version <br>
$ `qemu-system-arm --version` <br>
This guide uses version 6.2.0. If yours is lower, it may not work for you.

* This guide is for emulating Bullseye __Lite__ (No GUI). <br>
In theory, it should also work with Bullseye Desktop, but make sure to delete `-nographic` in the launch command.

* The internet connection within the VM is very slow (~50kB/s while running apt upgrade). <br>
This guide uses [USB as a network interface for the VM](https://stackoverflow.com/a/64420363/18271103). <br>
If possible, setting up a bridged connection or a NAT interface would probably be better.

* [There seems to be a bug](https://bugs.launchpad.net/ubuntu/+source/libvirt/+bug/1936831) which makes it impossible to use virt utils to manager this VM.

* Some steps in the VM's boot sequence fail, but they do not seem to affect functionality.

_I honestly do not recommend doing this, but if you insist, good luck._

## Steps
### 1. (Recommended) Create a working dirctory and change to it
* $ `sudo mkdir rpidir && cd rpidir`

### 2. Download compressed OS image
* $ `wget https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2022-09-26/2022-09-22-raspios-bullseye-armhf-lite.img.xz`
* (Or go to the [official download page](https://www.raspberrypi.com/software/operating-systems/).)

### 3. Extract OS image <br>
* $ `unxz 2022-09-22-raspios-bullseye-armhf-lite.img`

### 4. (Optional) Rename OS image
* $ `mv 2022-09-22-raspios-bullseye-armhf-lite.img bullseye.img` <br>
(The rest of this guide assumes it's renamed to bullseye.img)

### 5. Find out the device offset in the image
* $ `fdisk -l bullseye.img` <br>
Sample output:
```
Disk bullseye.img: 1.75 GiB, 1874853888 bytes, 3661824 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xb1214a26

Device        Boot  Start     End Sectors  Size Id Type
bullseye.img1        8192  532479  524288  256M  c W95 FAT32 (LBA)
bullseye.img2      532480 3661823 3129344  1.5G 83 Linux
```

* Then multiply the start address of the first device by the the sector size to obtain the offset. <br> `8192 * 512 = 4194304`

### 6. Create a mount point for the OS image
* $ `sudo mkdir /mnt/tmpmnt`

### 7. Mount the OS image at the mount point with the calculated offset
* $ `sudo mount -o loop,offset=4194304 bullseye.img /mnt/tmpmnt`

### 8. Copy the kernel from the mounted image
* $ `cp /mnt/tmpmnt/kernel8.img kernel.img` <br>
(Renamed to kernel.img for convenience)

### 9. Copy the dtb from the mounted image
* $ `cp /mnt/tmpmnt/bcm2710-rpi-3-b-plus.dtb treeblob.dtb` <br>
(Renamed to treeblob.dtb for convenience)

### 10. Generate an encrypted password
* $ `echo 'password' | openssl passwd -6 -stdin` <br>
(The output of this command is an encrypted version of the password, necessary for the next step)

### 11. Create and edit userconf.txt file in the mounted image
* $ `sudo nano /mnt/tmpmnt/userconf.txt`
* Type your crendentials in it, following the format `username:encrypted_password` .
* Press Ctrl+S to save and Ctrl+X to exit.

### 12. Unmount the OS image
* $ `sudo umount /mnt/tmpmnt`

### 13. Resize the OS image 
* $ `qemu-img resize bullseye.img 8G` <br>
(You may choose another size, but make sure it's a power of 2 and greater than the initial size)

### 14. Install qemu-system-arm
* $ `sudo apt install qemu-system-arm`

### 15. Start the VM using the following command

```
qemu-system-aarch64 \
	-machine raspi3b \
	-cpu cortex-a72 \
	-smp 4 -m 1G \
	-kernel kernel.img \
	-dtb treeblob.dtb \
	-drive "file=bullseye.img,format=img,index=0,if=sd" \
	-append "rw earlyprintk loglevel=8 console=ttyAMA0,115200 dwc_otg.lpm_enable=0 root=/dev/mmcblk0p2 rootdelay=1" \
	-device usb-net,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 \
	-nographic
```

* Make sure to edit the command according to your files' names.
* (Recommended: save the command as a shell script for easier/faster launch)

It's okay if you don't see anything in the terminal. Just wait. The VM should start booting. <br>
The boot sequence takes a while and some boot steps may fail, but that's probably fine. <br>
Once it's done, you'll be prompted to login. Enter the username and password you specified earlier. <br>
(Note: Some boot messages may keep popping up as you log in.)

## Once you're logged in
At this point, the VM is set up and usable. However, it is recommended that you do the following:

### 1. Resize the main partition to fill the available space

1. Start the cfdisk utility <br>
   * $ `sudo cfdisk /dev/mmcblk0` <br>
(/dev/mmcblk0 is the device file of the SD card on the Raspberry Pi 3. The cfdisk utility allows us to modify the partitions on the SD card)
   * In the utility, you can use the up and down arrow keys to navigate the partitions, and the side arrow keys to navigate the options.

2. Select the `/dev/mmcblk0p2` partition and __Resize__ it to fill the available space. <br>
(The default given size should be just fine.)

3. Save changes with __Write__, then __Quit__.

4. Resize the file system
   * $ `sudo resize2fs /dev/mmcblk0p2`

5. Shutdown and restart the VM
   * $ `sudo shutdown now`

### 2. (Recommended..?) Update the VM's system
* $ `sudo apt update && sudo apt upgrade` <br>
(Note: The internet connection within the VM is very slow, and this will take a long time.)

### 3. Use raspi-config to configure the system
* $ `sudo raspi-config` <br>
(This tool enables you to configure your locale, keyboard layout, hostname, username, password, as well as turn on SSH server, and more)

## To access the VM from the host machine using SSH
* Make sure that you have enabled the SSH server through the raspi-config utility.
* $ `ssh username@localhost -p 5555` <br>
(5555 being the forward port configured for the VM in the launch command.)
* If you intend to change the VM's ssh server port, make sure to change the forwarded port (from 22) in the launch command as well.

---

## Sources and additional information
* [Emulate Raspberry Pi with QEMU](https://azeria-labs.com/emulate-raspberry-pi-with-qemu/) on Azeria-Labs.com.
* [Emulating ARM64 Raspberry Pi Image using QEMU](https://youtu.be/Y-FUvi1z1aU) (Sep 2022) by Source Meets Sink.
* [System emulation using qemu: Raspberry PI 3](https://raduzaharia.medium.com/system-emulation-using-qemu-raspberry-pi-3-4973260ffb3e) (Sep 2022) by Radu Zaharia.
* (Password configuration) [An update to Raspberry Pi OS Bullseye](https://www.raspberrypi.com/news/raspberry-pi-bullseye-update-april-2022/) (Apr 2022) by Simon Long.
* More sources and information in [my Rasbian Buster emulation guide](https://gist.github.com/KhalilOuali/f74a5fec25457a383a8cd93147f868ca).

Thank you to the linux emulation community, and to the FOSS community as a whole.
