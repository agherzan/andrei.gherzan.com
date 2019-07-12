---
layout:	braindump
title: 64bit kernel on Raspberry Pi
description: Compile a 64 bit kernel for a supported Raspberry Pi board and deploy it on a Raspbian image
excerpt: Compile a 64 bit kernel for a supported Raspberry Pi board and deploy it on a Raspbian image
category: linux
image: /assets/img/rpi-logo.png
---

### TL;DR

- guide for compiling Raspberry Pi 64bit kernel, stubs (including toolchain and cross-compiler)
- guide for switching Raspbian from 32bit kernel to a 64bit one

# About

![Raspberry Pi Logo](/assets/img/rpi-logo.png){:class="image-right"}I should start this braindump by trying to address the question `Why?`. It is not an easy one. The performance impact is minimal. While maintaining the [Yocto Raspberry Pi BSP layer](https://github.com/agherzan/meta-raspberrypi), I have heard people saying that running video applications shows some noticeable performance gain - but that is with kernel and user-space on 64bit. This braindump doesn't address the user-space part as it is based on Raspbian. If you want to go full 64, probably the easiest way forward remains `Yocto/buildroot`. Otherwise consider this an exercise for running your benchmarks or giving 64bit kernel a try.

## A. Compile aarch64 toolchain

Let's first create a workspace where we will build our toolchain:

```sh
mkdir -p toolchains/aarch64
cd toolchains/aarch64
export TOOLCHAIN=`pwd` # Used later to reference the toolchain location
```

First of all, we will need to compile a set of `tools`. The first bit is [binutils](https://www.gnu.org/software/binutils/) which provides, in summary, a linker and an assembler. So we will download the current latest version (tested on 2.32) and compile it. Feel free to try a newer version. It shouldn't have any impact (fingers crossed).

```sh
cd "$TOOLCHAIN"
wget https://ftp.gnu.org/gnu/binutils/binutils-2.32.tar.bz2
tar -xf binutils-2.32.tar.bz2
mkdir binutils-2.32-build
cd binutils-2.32-build
../binutils-2.32/configure --prefix="$TOOLCHAIN" --target=aarch64-linux-gnu --disable-nls
make -j4
make install
```

Now you have `binutils` in place. The next step is a C compiler, [GCC](https://gcc.gnu.org/). We will be compiling a very minimal compiler for C that is enough for our use-case. Feel free though to modify the configuration line as wanted or even download a newer version if available.

```sh
cd "$TOOLCHAIN"
wget https://ftp.gnu.org/gnu/gcc/gcc-9.1.0/gcc-9.1.0.tar.gz
tar -xf gcc-9.1.0.tar.gz
mkdir gcc-9.1.0-build
cd gcc-9.1.0-build
```

`GCC` will have some dependencies so make sure you have them installed on your host. For example, on Ubuntu you will need:

```sh
sudo apt-get install libgmp-dev libmpfr-dev libmpc-dev
```

As soon as you have all the needed dependencies installed we can proceed with `GCC` compilation. If there are more dependencies missing just rerun configure until it finishes successfully.

```sh
../gcc-9.1.0/configure --prefix="$TOOLCHAIN" --target=aarch64-linux-gnu --with-newlib --without-headers --disable-nls --disable-shared --disable-threads --disable-libssp --disable-decimal-float --disable-libquadmath --disable-libvtv --disable-libgomp --disable-libatomic --enable-languages=c
make all-gcc -j4
make install-gcc
```

## B. Build the Raspberry Pi kernel

Before starting the compilation there are some packages we need on the host. Here is an example on Ubuntu:

```sh
sudo apt-get install bison flex
```

Same as above, if running the configure below fails asking for other dependencies, act accordingly: depending on your distro find the corresponding package and install it.

The current stable kernel branch (on the Raspberry Pi fork) is `rpi-4-19-y` so this is the one we will be using below. As development continues, modify the branch after checking the GitHub [repository](https://github.com/raspberrypi/linux).

Raspberry Pi 3 and 4 are both 64bit capable and thus have a specific `defconfig` available in the kernel sources. We will need the name of the `defconfig` when configuring the kernel below. These are not the only `defconfigs` that can be used but are the official, tested ones for these boards:

- for Raspberry Pi 3: `bcmrpi3_defconfig`
- for Raspberry Pi 4: `bcm2711_defconfig`

That said, let's compile a Raspberry Pi 64bit kernel:

```sh
git clone https://github.com/raspberrypi/linux.git rpi-linux
cd rpi-linux
git checkout origin/rpi-4.19.y # change the branch name for newer versions
mkdir kernel-build
PATH=$PATH:$TOOLCHAIN/bin make O=./kernel-build/ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-  thiswontwork_defconfig # change the name of the defconfig with the one for the targeted board
PATH=$PATH:$TOOLCHAIN/bin make -j4 O=./kernel-build/ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
export KERNEL_VERSION=`cat ./kernel-build/include/generated/utsrelease.h | sed -e 's/.*"\(.*\)".*/\1/'` # we extract and export the kernel version to be able to correctly deploy the modules below when deploying them on the Raspbian image
make -j4 O=./kernel-build/ DEPMOD=echo MODLIB=./kernel-install/lib/modules/${KERNEL_VERSION} INSTALL_FW_PATH=./kernel-install/lib/firmware modules_install
export KERNEL_BUILD_DIR=`realpath kernel-build` # used if you want to deploy it to Raspbian, ignore otherwise
```

This will get us a kernel image, the associated kernel modules and a device tree.

## C. Deploy the compiled 64bit kernel on a Raspbian image

You now have all the bits to proceed in modifying a Raspbian image to boot a Raspberry Pi board in 64bit mode. First thing first, download Raspbian and burn it with any tool you prefer. I'd use [etcher](https://www.balena.io/etcher/). Mount the boot partition and copy kernel image and the kernel modules on it. The following commands assume that the boot partition mount point is `/run/media/me/boot`i and the root filesystem mount point is `/run/media/me/rootfs`. Also, the paths rely on the exports set throughout this braindump.

```sh
cp $KERNEL_BUILD_DIR/arch/arm64/boot/Image /run/media/me/boot/kernel8.img
cp $KERNEL_BUILD_DIR/arch/arm64/boot/dts/broadcom/*.dtb /run/media/me/boot/
cp -r $KERNEL_BUILD_DIR/kernel-install/* /run/media/me/rootfs/
```

***Be aware that Raspberry Pi 4 needs some additional quirks which are detailed in [this](https://andrei.gherzan.ro/linux/raspbian-rpi4-64/) braindump. This will be addressed soon but for now make sure you carry on those additional changes.***

```sh
sync
umount /run/media/me/boot /run/media/me/rootfs
```

Pop the microSD card in your board, attach a serial console, boot, login and

```sh
uname -a
Linux raspberrypi 4.19.56-v8+ #1 SMP PREEMPT Thu Jul 4 12:43:20 BST 2019 aarch64 GNU/Linux
```

At first boot all the kernel modules will fail loading. This is because the kernel modules dependency files are not generated. You will need to run `depmod`.

```sh
sudo depmod -a
sudo reboot
```

At next boot everything should be up and running, that is a Raspbian 32bit user-space on a 64bit linux kernel.
