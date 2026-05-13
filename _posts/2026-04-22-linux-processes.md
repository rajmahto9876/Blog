---
title: Cross Compiler (beaglebone)
date: 2026-04-22
categories: [Linux-Driver-Development]
tags: [linux, Setup, Cross-Compiler]
---

# Cross Compiling Linux Kernel for beaglebone Black
Building and compiling the Linux kernel for embedded systems is one of the most important skills in Embedded Linux development. In this blog, we will walk through the complete process of:

- Setting up dependencies
- Downloading the ARM cross compiler
- Fetching beaglebone Linux source
- Building Kernel Image and Device Trees
- Building Kernel Modules
- Fixing common compilation issues


# Why Cross Compilation?

beaglebone Black uses an ARM processor, while most development machines use x86_64 architecture. Instead of compiling directly on the board (which is slow), we compile on the host machine using an ARM cross compiler.

This process is called:
> **Cross Compilation**

---

# System Requirements

## Install Required Dependencies

```bash
sudo apt install build-essential libncurses-dev gawk flex bison \
openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev \
libiberty-dev autoconf llvm lzop u-boot-tools
```

# Download ARM Cross Compiler
Install Linaro ARM toolchain for cross compilation.

---
# Toolchain Download
Download the ARM GNU Toolchain from the official Linaro release page:

```link
https://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/arm-linux-gnueabihf/
```
or 

```bash
wget https://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/arm-linux-gnueabihf/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
```
---

# Compiler Package
```text
gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
tar -xf gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
```
# Add Toolchain to PATH
```bash
export PATH=$PATH:$PWD/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin
```
To make it permanent:

```bash
echo 'export PATH=$PATH:$HOME/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin' >> ~/.bashrc
source ~/.bashrc
```

# Verify Installation
Check if the compiler is properly installed or not

```bash
arm-linux-gnueabihf-gcc --version
```
Expected output:
```bash
arm-linux-gnueabihf-gcc (Linaro GCC 7.5-2019.12) 7.5.0
```

# Connect to beaglebone
```bash
ssh -x debian@192.168.7.2
```

Check running kernel version:
```shell
uname -r
```

Output:
```shell
4.19.94-ti-r42
```

# Clone beaglebone Linux Repository
The beaglebone Linux kernel is maintained here:

```shell
git clone https://github.com/beagleboard/linux.git
cd linux
git checkout 4.19.94-ti-r42
```

# Compilation Steps

## Step 1: Clean Previous Builds :
Removes all the temporary folder, object files, images    generated during the previous build. This step also deletes the .config file
if created previously.
```shell
make ARCH=arm distclean
```
## Step 2: Load Default beaglebone Configuration
This creates a default kernel configuration file (.config)
```shell
make ARCH=arm bb.org_defconfig
```
## Step 3: Optional Kernel Configuration <br>
Use this step if you want to:
Enable drivers, Disable features, Modify kernel options

```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```

## Step 4: Build Kernel Image and Device Trees

This stage compiles: Linux Kernel, Device Tree Blobs (DTBs), Kernel image (uImage)
```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- uImage dtbs LOADADDR=0x80008000 -j4
```
Understanding Build Parameters

| Parameter | Description |
|------------|-------------|
| `ARCH=arm` | Specifies the target architecture as ARM. |
| `CROSS_COMPILE=` | Prefix used for the ARM cross-compilation toolchain. |
| `uImage` | Generates a U-Boot compatible Linux kernel image. |
| `dtbs` | Compiles Device Tree Source files into Device Tree Blobs (`.dtb`). |
| `LOADADDR` | Defines the kernel load address in memory. |
| `-j4` | Enables parallel compilation using 4 CPU cores for faster builds. |

## Step 5: Build Kernel Modules
This builds all loadable kernel modules (.ko files).
```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules -j4
```

## Step 6: Install Kernel Modules
This installs modules into: /lib/modules/<kernel-version>
```shell
sudo make ARCH=arm modules_install
```

``` shell
Expected Output: After successful compilation, important outputs include:

Output	Description
uImage	Linux kernel image
zImage	Compressed kernel image
*.dtb	Device Tree Blobs
*.ko	Kernel modules
Common Compilation Error
```
# Fix Error (specifically for this branch)

Sometimes kernel compilation fails because of:
scripts/dtc/dtc-lexel.lex.c

Error occurs near:
```c
YYLTYPE yylloc;

Fix for Compilation Error
File
scripts/dtc/dtc-lexel.lex.c
Around Line 629
Replace:
YYLTYPE yylloc;
With:
extern YYLTYPE yylloc;
```
