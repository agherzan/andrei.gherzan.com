---
layout:	braindump
title: U-Boot on Raspberry Pi (including RPi 4)
description: Chain U-Boot in the boot process of a Raspberry Pi (including Raspberry Pi 4)
excerpt: Chain U-Boot in the boot process of a Raspberry Pi (including Raspberry Pi 4)
category: linux
image: /assets/img/rpi-logo.png
---

### TL;DR

- guide for compiling u-boot for Raspberry Pi boards
- guide for chaining u-boot in the boot process of Raspbian

## About

[U-Boot](https://www.denx.de/wiki/U-Boot), `Das U-Boot`, is a very common bootloader in embedded systems. It includes support for various computer architectures and, following this guide, you will be able to run it on your Raspberry Pi.

## Toolchain

First of all you will need some tools, including a cross compiler, to be able to proceed with compiling this component. There are a set of options in this regards.

### Download a toolchain

You can download one from various online providers. One example, probably the most common one, is [Linaro](https://www.linaro.org/downloads/). Download the right toolchain for your target. For example if you want to compile a U-Boot for a 32b Raspberry Pi, download the `arm-linux-gnueabihf` toolchain. Dearchive the file and set a variable so we can point the compilation to this toolchain:

```sh
tar -xf gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf.tar.xz
export TOOLCHAIN="$(pwd)/gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf/bin"
```

### Compile your own toolchain

I wrote a braindump which includes instructions for doing this for a 64bit toolchain. See [this](https://andrei.gherzan.ro/linux/raspbian-rpi-64/) for more information. You will need to tweak things around for a 32bit one but things should stay pretty similar. Using that guide or a similar one you should end up with a `TOOLCHAIN` variable definition for the `bin` directory of the toolchain.

### Yocto

You can also let [Yocto](https://www.yoctoproject.org/) deal with this but that is not the target of this guide. In summary, all you need to do is setup a build with the appropriate `MACHINE` set. The Raspberry Pi machines are defined in [meta-raspberrypi](https://github.com/agherzan/meta-raspberrypi).

## Compile U-Boot

We will be using version `v2019.07` in this guide. [Upstream](https://gitlab.denx.de/u-boot/u-boot) has support for all the Raspberry Pi boards with the exception of Raspberry Pi 4 for which you will need to use a fork until support gets upstreamed. As the fork inherits the support for all the boards, we will be using it in the following commands.

```sh
git clone https://github.com/agherzan/u-boot.git u-boot
cd u-boot
git checkout ag/v2019.07-rpi4-wip
```

There is also a backport branch for `v2019.01`: [ag/v2019.01-rpi4-wip](https://github.com/agherzan/u-boot/tree/ag/v2019.01-rpi4-wip).

We have the code, now we can proceed to compilation. We will be using different defconfigs based on the targeted board as it follows:

- Raspberry Pi 3 (32b): `rpi_3_32b_defconfig`
- Raspberry Pi 3 (64b): `rpi_3_defconfig`
- Raspberry Pi 4 (32b): `rpi_4_32b_defconfig`
- Raspberry Pi 4 (64b): `rpi_4_defconfig`

Also, you will need the name of the toolchain. This is the `triplet` of the tools available in the toolchain's bin directory. For example if you toolchain includes the `ar` archiver as a file called `arm-linux-gnueabihf-ar`, the `CROSS_COMPILE` variable used by U-Boot compile process will be `CROSS_COMPILE=$TOOLCHAIN/arm-linux-gnueabihf-`. Make sure you adapt this variable below for other toolchains.

```sh
CROSS_COMPILE=$TOOLCHAIN/arm-linux-gnueabihf- make pick_the_right_defconfig
CROSS_COMPILE=$TOOLCHAIN/arm-linux-gnueabihf- make -j8
```

This is most probably obvious but it doesn't hurt to be mentioned. The `defconfig` used above needs to match the toolchain and in turn the triplet used for `CROSS_COMPILE` variable. This will break if for example you are trying to build a `rpi_3_32b_defconfig` with a `aarch64` toolchain.

To make the boot process automatic let's compile a user-defined image file that will setup the `bootcmd` in U-Boot:

```sh
$ cat << EOF > boot.cmd
fdt addr \${fdt_addr} && fdt get value bootargs /chosen bootargs
fatload mmc 0:1 \${kernel_addr_r} uImage
bootm \${kernel_addr_r} - \${fdt_addr}
EOF
$ mkimage -A arm -T script -C none -n "Boot script" -d "boot.cmd" boot.scr
```

You will need `mkimage` available on your host. On `Archlinux`, for example, this tool is provided by [uboot-tools](https://www.archlinux.org/packages/community/x86_64/uboot-tools/). Have some research for your distro.

With that being done all we need now is to define a variable so we can reference it when deploying the compiled bits:

```
export UBOOT_BUILD_DIR="$(pwd)"
```

## Run Raspbian with U-Boot

You now have all the required elements to proceed in modifying a Raspbian image to boot a Raspberry Pi board with U-Boot. First thing first, download Raspbian and burn it with any tool you prefer. I'd use [etcher](https://www.balena.io/etcher/). Mount the boot partition and proceed with copying the compiled U-Boot and other changes. The following commands assume that the boot partition mount point is `/run/media/me/boot`. Also, the paths rely on the exports set throughout this braindump.

The Raspberry Pi bootloader selects a kernel image from the boot partition based on the board it boots:

- `kernel8` for Raspberry Pi 3 and 4 in 64 bit mode
- `kernel7l.img` for Raspberry Pi 4 in 32bit mode (with LPAE)
- `kernel7.img` for Raspberry Pi 2, 3 and 4 in 32bit mode (no LPAE)
- `kernel.img` for the rest

Let's define the one you are targeting so the next commands can simply use a variable:

```
export KERNEL=kernel7l.img
```

We need to convert this image into one that U-Boot can load with `bootm` (as we set the `boot.cmd` file):

```
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n "Linux kernel" -d /run/media/me/boot/$KERNEL /run/media/me/boot/uImage
```

All is left now is to copy U-boot, make a small `config.txt` change and `umount`:

```sh
cp $UBOOT_BUILD_DIR/u-boot.bin /run/media/me/boot/$KERNEL
cp $UBOOT_BUILD_DIR/boot.scr /run/media/me/boot
echo "enable_uart=1" >> /run/media/me/boot/config.txt
sync
umount /run/media/me/boot
```

The `enable_uart` setting is required because U-Boot assumes the `VideoCore` firmware is configured to use the mini UART (rather than `PL011`) for the serial console. Without this, U-Boot will not boot at all.

Pop the microSD card in your board, attach a serial console and boot:

```plain
MMC:   emmc2@7e340000: 0
Loading Environment from FAT... *** Warning - bad CRC, using default environment

In:    serial
Out:   serial
Err:   serial
Net:   No ethernet found.
starting USB...
Bus usb@7e980000: scanning bus usb@7e980000 for devices... 1 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found
Hit any key to stop autoboot:  0 
switch to partitions #0, OK
mmc0 is current device
Scanning mmc 0:1...
Found U-Boot script /boot.scr
213 bytes read in 34 ms (5.9 KiB/s)
## Executing script at 02400000
5608976 bytes read in 1165 ms (4.6 MiB/s)
## Booting kernel from Legacy Image at 00080000 ...
   Image Name:   Linux kernel
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    5608912 Bytes = 5.3 MiB
   Load Address: 00008000
   Entry Point:  00008000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 2eff6000
   Booting using the fdt blob at 0x2eff6000
   Loading Kernel Image ... OK
   Using Device Tree in place at 2eff6000, end 2f002f5a

Starting kernel ...
[...]
```

The output will be different depending on your board but not too much. The Raspberry Pi will load U-Boot which in turn will load and run the kernel and you will end up in Raspbian.
