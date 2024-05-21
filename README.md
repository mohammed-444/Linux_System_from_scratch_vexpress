# Embedded Linux System for Vexpress_ca9x4

This repository contains the necessary components to build an embedded Linux system from scratch, including libraries, tools, and custom configurations for Vexpress_ca9x4.

## Components

- [**Crosstool-NG**](#crosstool-ng): A toolchain generator for building cross-compiling toolchains.
- [**Emulated SD Card**](#emulated-sd-card): make a virtual SD card to load from 
- [**U-Boot**](#u-boot): A boot loader used in embedded systems.
- [**Kernel**](#kernel): The Linux kernel for embedded systems.
- [**U-Boot config to read the kernel dtb and rootfs**](#u-boot-config-to-read-the-kernel-dtb-and-rootfs)
- [**BusyBox**](#busybox): A collection of Unix utilities in a single executable for embedded systems.
- [**Rootfs**](#rootfs): build the folder and set init process which is the first process started during bootup, responsible for initializing the system.


## Usage

To use this repository, follow the steps below:

1. Clone the repository to your local machine.
2. Install the necessary dependencies for building the system, such as `gcc`, `make`, and `libssl-dev`.
3. Follow the instructions in the tool's documentation to configure and build the system.

---
---
## Crosstool-NG

Crosstool-ng is an open-source tool designed to simplify and automate the process of building cross-compilation toolchains. Cross-compilation  toolchains are essential for developing software that runs on a different architecture or platform than the one on which it is compiled. This is common in embedded systems development, where software is often compiled on a host machine but executed on a target device with different hardware architecture.

- ([**Documentation**](https://crosstool-ng.github.io/docs/)) 
- ([**Repo**](https://github.com/crosstool-ng/crosstool-ng))

### Download crosstool-ng

```
git clone https://github.com/crosstool-ng/crosstool-ng.git
git checkout 25f6dae8
cd crosstool-ng/
```

### Setting up Enviornment

Need to download the following dependencies

```shell
sudo apt-get install -y gcc g++ gperf bison flex texinfo help2man make libncurses5-dev \
python3-dev autoconf automake libtool libtool-bin gawk wget bzip2 xz-utils unzip \
patch libstdc++6 rsync
```

```
./bootstrap #Run bootstrap to setup all enviornment
```

```
./configure --enable-local #To check all dependency
```

```
make #To generate the Makefile for croostool-ng
```

### Configuration for Crosstool-ng 

```
#To list all microcontrollers supported
./ct-ng list-samples
```



We will choose `arm-cortexa9_neon-linux-gnueabihf` :

```
#To configure the microcontroller used
./ct-ng arm-cortexa9_neon-linux-gnueabihf
```

```
#To configure our toolchain
./ct-ng menuconfig
```


**Our Needed Configurations :**

* [ ] C-library : **musl**
* [ ] C compiler : **support C++**
* [ ] Companion tools : **autoconf , automake, make**
* [ ] Debug facilities : **starce** , **gdb**

```
#Will show our configurations
./ct-ng show-config
```



### Build Our ToolChain
**take some time to run**
```
#To build and get your toolchain
./ct-ng build
```

Wait till it finish and go check your tool chain

```
cd ~/x-tools
```



### Delete the Toolchain menuconfig if you want to start over again
```
./ct-ng distclean
```
---
---
## Emulated SD Card

In this section it's required to have SD card with first partition to be FAT as pre configured in **U-boot Menuconfig**.
In order to Emulate SD card to attached to Qemu following steps will be followed: (ONLY FOR NON PHYSICAL SD):

```bash
# Create a file with 1 GB filled with zeros in the place you want
dd if=/dev/zero of=sd.img bs=1M count=1024 status=progress 

# Create a folder to mount the device to 
mkdir -p SD/{boot,rootfs}
```

### Configure the Partition Table for the SD card

```bash
# for the VIRTUAL SD card
cfdisk sd.img

```
choose `dos` & `primary`
| Partition Size | Partition Format | Bootable  |
| :------------: | :--------------: | :-------: |
|    `256 MB`    |     `FAT 16`     | ***Yes*** |
|    `767 MB`    |     `Linux`      | ***No***  |

**write** to apply changes

### Loop Driver FOR Virtual usage 

To emulate the sd.img file as a sd card we need to attach it to **loop driver** to be as a **block storage**

```bash
# attach the sd.img to be treated as block storage
sudo losetup -f --show --partscan sd.img

# Running the upper command will show you
# Which loop device the sd.img is connected
# take it and assign it like down bellow command

# Assign the Block device as global variable to be treated as MACRO
export DISK=/dev/loop<x>
```

### Format Partition Table

As pre configured from **cfdisk command** first partition is **FAT**

```bash
# Formating the first partition as FAT
sudo mkfs.vfat -F 16 ${DISK}p1
```

 As pre configured from cfdisk Command second partition is linux

```bash
# format the created partition by ext4
sudo mkfs.ext4 ${DISK}p2
```
mount to a folder
```bash
sudo mount ${DISK}p1 SD/boot
sudo mount ${DISK}p2 SD/rootfs
```
you can check 
```bash
sudolsblk | grep grep loop[0-9]*p[1-2]

```
 * after you close the pc the loop virtual device will be deleted
 * you need to create it again [Loop Driver FOR Virtual usage  ](#Loop-driver-for-virtual-usage) 
 * you do not need to run [mkfs again](#format-partition-table) you only need to use mkfs for the first time to make the file system
 * then mount them again
now you made virtual SDcard to use in the next step

---
---
## U-boot

U-Boot is **the most popular boot loader in** **linux** **based embedded devices**. It is released as open source under the GNU GPLv2 license. It supports a wide range of microprocessors like MIPS, ARM, PPC, Blackfin, AVR32 and x86.



### Download U-Boot

```bash
git clone git@github.com:u-boot/u-boot.git
cd u-boot/
```

### Configure U-Boot 

In this section we will **configure** u-boot for the Machine

```bash
# In order to find the machine supported by U-Boot
ls ./configs/ | grep vexpress_ca9x4  #[your machine] 
# ls configs | grep vexpress*
```

### Vexpress Cortex A9 (Qemu)

In **U-boot directory** Assign this value

```bash
# Set the Cross Compiler into the environment
# To be used by the u-boot
# Compiler out from Crosstool-NG
export CROSS_COMPILE=<Path To the Compiler>/arm-cortexa9_neon-linux-musleabihf-
export ARCH=arm

# load the default configuration of ARM Vexpress Cortex A9
make vexpress_ca9x4_defconfig
```


### Configure U-Boot 

In this part we need to configure some u-boot configuration for the specific board chosen up.

```bash
make menuconfig
```

**The requirement are like following for using virtual SD**:

- [ ] Support **editenv** search for **CMD_EDITENV** .
- [ ] Support **bootd** search for **CMD_BOOTD** to enbale the run from memory card
- [ ] Unset support of Flash search for **ENV_IS_IN_FLASH** 
- [ ] Support FAT file system search for **ENV_IS_IN_FAT** 
  - [ ] Configure the FAT interface to **mmc** search for **ENV_FAT_INTERFACE** so when I use mmc in the u-boot cmdline it will map the to virtual SDcard 
  - [ ] Configure the partition where the fat is store to **0:1** search for **ENV_FAT_DEVICE_AND_PART** used to know which partition store the fat files system

### build u-boot
```bash
# number of core to use to run
make -j 8
```
## Test U-Boot (can be skipped)

Check the **u-boot** and the **sd card** are working

### Vexpress-a9 (QEMU)

Start Qemu with the **Emulated SD** card

```bash
qemu-system-arm -M vexpress-a9 -m 128M -nographic \
-kernel u-boot/u-boot \
-sd sd.img
```

### Load File to RAM test inside the u-boot cmdline

First we need to know the ram address by running the following commend

```bash
# this commend will show all the board information and it start ram address
bdinfo
```
```bash
# show the file system content
ls mmc 0:1 
```

### Load from FAT

```bash
# addressRam is a variable knowen from bdinfo commend
fatload mmc 0:1 [addressRam] [fileName]
```

## Kernel


go to `https://www.kernel.org/` and download the kernel version you want to use as  `filename.tar.xz`
Extract downloaded archives:
```
tar xf filename.tar.xz
# then enter the directory
cd filename
```
you can find deconfig for arm under `./arch/arm/configs/`

### Set deconfig for vexpress  :

```
# Set the Cross Compiler into the environment
# To be used by the kernel
# Compiler out from Crosstool-NG

export CROSS_COMPILE=<Path To the Compiler>/arm-cortexa9_neon-linux-musleabihf-
export ARCH=arm

make vexpress_defconfig

```
### change in configuration (if needed )

```

make menuconfig

```

### Compile actual kernel

takes some time depend on you machine
```

make -j8 zImage modules dtbs  

```

```
export INSTALL_MOD_PATH=<Path To the lib dir>
sudo make modules_install 
```

the output zImage you can found under 
* `./arch/arm/boot/zImage`
* `./arch/arm/boot/dts/arm/vexpress-v2p-ca9.dtb`
---
---
### download the image & dtb 

```
sudo cp -rv ./arch/arm/boot/zImage <Path_To_the_SDcard>/SD/boot
sudo cp -rv ./arch/arm/boot/dts/arm/vexpress-v2p-ca9.dtb <Path_To_the_SDcard>/SD/boot
```
---
---
## U-Boot config to read the kernel dtb and rootfs

firt you must know that the u-boot run `bootcmd` automatically if know one interfere 

### run qemu to start the config
```
qemu-system-arm -M vexpress-a9 -m 128M -nographic \
-kernel u-boot/u-boot \
-sd sd.img
```
### start the configs inside u-boot cmdline
```
# var when run load the kernel to ram 

setenv LOAD_FAT "fatload mmc 0:1 $kernel_addr_r zImage;fatload mmc 0:1 $fdt_addr_r vexpress-v2p-ca9.dtb";

# var when run start the kernel

setenv start_k "bootz $kernel_addr_r - $fdt_addr_r";

# let's set the auatomatically called varialbe so when called again do as we wants

setenv bootcmd "run LOAD_FAT; run start_k"

# the kernel uses this variable to know where to look for the rootfs (later will be used)

setenv bootargs \"console=ttyAMA0,115200n8 root=/dev/mmcblk0p2 rootfstype=ext4 rw rootwait init=init\";


# then final step let's save all this change to boot.env(will be created autmatically in the boot folder) 

saveenv
```
--- 
---
## BusyBox

### Download repo

```

git clone git://busybox.net/busybox.git --branch=1_33_0 --depth=1
cd BusyBox

````
you can find deconfig for arm under `./arch/arm/configs/`

# Set deconfig  :

```
# Set the Cross Compiler into the environment
# To be used by the busybox
# Compiler out from Crosstool-NG

export CROSS_COMPILE=<Path To the Compiler>/arm-cortexa9_neon-linux-musleabihf-
export ARCH=arm

make defconfig 
```
### change in configuration 

```

make menuconfig

Settings -> Build static binary 	(no shared libraries) 	Enable
Settings -> Destination path for ‘make install’ 	(if you want)

make -j8 
make install

```
it will be install under `./_install`
after this step we succeed to build the binary for the rootfs

---
---
## Rootfs
What do we want for our minimalistic Linux system? Only a few things:

   - init script – Kernel needs to run something as first process in the system.
   - Busybox – it will contain basic shell and utilities (like cd, cp, ls, echo etc)
  

let's create the folder structure first for rootfs:
---
```
# inside the SD/rootfs 
sudo mkdir -pv rootfs/{bin,sbin,etc,proc,sys,dev,usr/{bin,sbin}}

# copy the binary from under the busybox to this folder 
sudo cp -av <Your_Path>busybox/_install/* <Your_Path>SD/rootfs/

cd rootfs # every thing we do under it now  <--------------
```

make the init script 
---
```
# generate a simbolic link to busybox and it will handle the init process for you

sudo ln -sf bin/busybox init 

sudo chmod +x init
```
creat rcS to run in the init process
---
it will run to mount the folder to the kernel `/proc` `/sys` and make the special files in `/dev`
```
sudo mkdir etc/init.d
sudo touch etc/init.d/rcS
sudo chmod +x etc/init.d/rcS
```
```
Add the following entries to etc/init.d/rcS:

#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys

mdev -s  # -s "Scan /sys and populate /dev"

echo "you are in user space rootfs has been uploaded "

```
# finally run this qemu to see the project
```bash
qemu-system-arm -M vexpress-a9 -m 128M -nographic \
-kernel u-boot/u-boot \
-sd sd.img
```
---
---
## U-boot Commands to add knowlage (can be skipped)

### Write Script in U-boot

```bash
setenv ifcondition "if mmc dev; then echo MmcExist; elif mtd list; then echo NoDeviceExist; else echo noDeviceExist; fi"
```

### help Command
	- ==> help
### U-Boot information commands
	- Version details: 
		- ==> version
	- NAND flash information:
		- ==> nand info
	- MMC information:
		- ==> mmc info
	- Board information:
		- ==> bdinfo
### U-Boot environment commands
	- Shows all variables
		- ==> printenv
	- Shows the value of a variable
		- ==> printenv <variable-name>
	- Changes the value of a variable or defines a new one, only in RAM
		- ==> setenv <variable-name> <variable-value>
	- Interactively edits the value of a variable, only in RAM
		- ==> editenv <variable-name>
	- Saves the current state of the environment to storage for persistence.
		- ==> saveenv
	- env command, with many sub-commands: env default, env info, env erase,
env set, env save, etc.

### U-Boot Commands to inspect or modify any memory location
	- Memory display
		- ==> md [.b, .w, .l, .q] address [# of objects]
	- Memory write
		- ==> mw [.b, .w, .l, .q] address value [count]
	- Memory modify (modify memory contents interactively starting from address)
		- ==> mm [.b, .w, .l, .q] address

### U-Boot filesystem storage commands
	- #### U-Boot has support for many filesystems
		- The exact list of supported filesystems depends on the U-Boot configuration
	- ####  Per-filesystem commands
		- FAT: fatinfo, fatls, fatsize, fatload, fatwrite
		- ext2/3/4: ext2ls, ext4ls, ext2load, ext4load, ext4size, ext4write
		- Squashfs: sqfsls, sqfsload
	- ####  “New” generic commands, working for all filesystem types
		- Load a file: load <interface> [<dev[:part]> [<addr> [<filename> [bytes [pos]]]]]
		- List files: ls <interface> [<dev[:part]> [directory]]
		- Get the size of a file: size <interface> <dev[:part]> <filename> (result stored in filesize environment variable)
		- interface: mmc, usb
		- dev: device number, 0 for first device, 1 for second device
		- part: partition number


### Environment variables can contain small scripts, to execute several commands and test the results of commands.
	- Useful to automate booting or upgrade processes
	- Several commands can be chained using the ; operator
	- Tests can be done using if command ; then ... ; else ... ; fi
	- Scripts are executed using run <variable-name>
	- You can reference other variables using ${variable-name}
	- #### Examples
	'''
	- setenv bootcmd 'tftp 0x21000000 zImage; tftp 0x22000000 dtb; bootz
	0x21000000 - 0x22000000'
	- setenv mmc-boot 'if fatload mmc 0 80000000 boot.ini; then source; else
	if fatload mmc 0 80000000 zImage; then run mmc-do-boot; fi; fi'
	'''
### U-Boot booting commands
- #### Commands to boot a Linux kernel image
	- bootz → boot a compressed ARM32 zImage
	- booti → boot an uncompressed ARM64 or RISC-V Image
	- bootm → boot a kernel image with legacy U-Boot headers
	- zboot → boot a compressed x86 bzImage
- #### bootz [addr [initrd[:size]] [fdt]]
	- addr: address of the kernel image in RAM
	- initrd: address of the initrd or initramfs, if any. Otherwise, must pass -
	- fdt: address of the Device Tree passed to the Linux kernel
- #### Important environment variables
	- bootcmd: list of commands executed automatically by U-Boot after the count down
	- bootargs: Linux kernel command line

### Generic Distro boot
- kernel_addr_r: address in RAM to load the kernel image
- ramdisk_addr_r: address in RAM to load the initramfs image (if any)
- fdt_addr_r: address in RAM to load the DTB (Flattened Device Tree)
- pxefile_addr_r: address in RAM to load the configuration file (usually extlinux.conf)
- bootfile: the path to the configuration file, for example /boot/extlinux/extlinux.conf


