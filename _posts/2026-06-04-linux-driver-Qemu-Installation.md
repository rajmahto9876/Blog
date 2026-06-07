---
title: QEMU Installation
date: 2026-06-04
categories: [Linux-Driver-Development]
tags: [linux, Qemu Installation]
---

---
# Introduction
QEMU (Quick Emulator) is a machine emulator and virtualizer that can emulate various hardware platforms. In the context of embedded Linux development, it allows to simulate ARM processors and custom boards on the host PC. Whether working on kernel testing, device driver development, boot debugging, or filesystem testing, QEMU provides a safe, isolated environment to experiment. 

---

## System Architecture(QEMU Runs Linux)
### High-Level Block Diagram

```
+------------------------------------------------------+
|                     HOST PC                          |
|------------------------------------------------------|
| Ubuntu / Linux Host                                  |
|                                                      |
|  +-----------------------------------------------+   |
|  |                 QEMU Emulator                 |   |
|  |-----------------------------------------------|   |
|  |                                               |   |
|  |  Virtual ARM CPU                              |   |
|  |  Virtual RAM                                  |   |
|  |  Virtual UART                                 |   |
|  |  Virtual Network                              |   |
|  |  Virtual Block Device                         |   |
|  |                                               |   |
|  +--------------------+--------------------------+   |
|                       |                              |
+-----------------------|------------------------------+
                        |
                        v
              +-------------------+
              | Linux Kernel      |
              |  zImage/Image     |
              +-------------------+
                        |
                        v
              +-------------------+
              | Root Filesystem   |
              | BusyBox / Debian  |
              +-------------------+
                        |
                        v
              +-------------------+
              | User Applications |
              +-------------------+
```

### Requirement to Boot Linux on QEMU
A complete Linux system requires at least two components:
**1. Linux Kernel Image**
- `zImage` — Compressed ARM kernel image
- `Image` — Uncompressed ARM64 kernel image
- `uImage` — U-Boot wrapped kernel image

**2. Root Filesystem (RootFS)**
- `rootfs.ext4` — Filesystem image containing all system files
- `busybox.img` — Minimal filesystem with BusyBox utilities
- `debian.img` — Full Debian filesystem

---

## Phase 1: Installing QEMU
### Ubuntu / Debian Installation

First, update your package list and install QEMU along with essential development tools:

```bash
sudo apt update

sudo apt install qemu-system-arm \
                 qemu-system-aarch64 \
                 qemu-utils \
                 qemu-efi \
                 qemu-user-static \
                 qemu-kvm \
                 qemu-system-gui \
                 gcc-arm-linux-gnueabihf \
                 gcc-aarch64-linux-gnu \
                 build-essential \
                 libncurses-dev \
                 flex bison \
                 bc \
                 u-boot-tools \
                 git \
                 device-tree-compiler
```

### Verify Installation

Check that QEMU is installed correctly:

```bash
qemu-system-arm --version
```

### Check Available Boards
QEMU supports many different virtual boards. List them:

```bash
qemu-system-arm -machine help
```

Expected: `versatilepb`, `vexpress-a9`, `virt`, etc.

---

## Phase 2: Linux Boot Flowchart 

```
START
  |
  v
Launch QEMU
  |
  v
QEMU Creates Virtual Hardware
  |
  v
Load Kernel Image (zImage)
  |
  v
CPU Starts at Reset Vector
  |
  v
Kernel Decompression
  |
  v
start_kernel()
  |
  v
Memory Initialization
  |
  v
Interrupt Initialization
  |
  v
Driver Initialization
  |
  v
Mount Root Filesystem
  |
  v
Execute init Process
  |
  v
Launch Shell / User Space
  |
  v
SYSTEM READY
```

---

## Method 1: Running Pre-Built Linux on QEMU (Quickest Way)
### Step 1: Download Prebuilt Kernel

```bash
wget https://github.com/dhruvvyas90/qemu-rpi-kernel/raw/master/kernel-qemu-4.19.50-buster
```

### Step 2: Download Root Filesystem

```bash
wget https://people.debian.org/~aurel32/qemu/armhf/rootfs.ext4.gz
gunzip rootfs.ext4.gz
```

### Step 3: Boot Linux

Now launch QEMU with these images:

```bash
qemu-system-arm \
-M versatilepb \
-m 256M \
-kernel kernel-qemu-4.19.50-buster \
-drive file=rootfs.ext4,format=raw \
-append "root=/dev/sda rw console=ttyAMA0" \
-net nic \
-net user \
-nographic
```

### Expected Output

```
Booting Linux on physical CPU 0x0
Linux version 4.19...
Mounted root filesystem
Debian GNU/Linux login:
```

### Login Credentials

```
Username: root
Password: root
```

---

## Method 2: Build  Linux Kernel

### Step 1: Clone the Linux Kernel Repository

```bash
git clone https://github.com/torvalds/linux.git
cd linux
```

### Step 2: Configure Kernel for ARM QEMU

```bash
make ARCH=arm versatile_defconfig
```

Alternatively, for more control:

```bash
make ARCH=arm menuconfig
```

**Important Kernel Configurations for QEMU:**

Navigate to:
```
Device Drivers
 └── Virtio drivers
      [*] Virtio block driver
      [*] PCI driver for virtio devices
      [*] Virtio MMIO driver
```

These ensure your kernel can access virtual block devices and memory-mapped I/O.

### Step 3: Compile the Kernel

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)
```

The compiled kernel will be at:
```
arch/arm/boot/zImage
```

For ARM64:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

Kernel image location:
```
arch/arm64/boot/Image
```

### Step 4: Boot Your Custom Kernel

```bash
qemu-system-arm \
-M versatilepb \
-m 256M \
-kernel arch/arm/boot/zImage \
-drive file=rootfs.ext4,format=raw \
-append "root=/dev/sda rw console=ttyAMA0" \
-nographic
```

---

## Creating a Minimal BusyBox Root Filesystem

### Step 1: Install Build Tools

```bash
sudo apt update
sudo apt install build-essential \
                 gcc-aarch64-linux-gnu \
                 libncurses-dev \
                 bison \
                 flex \
                 bc \
                 wget
```

### Step 2: Download and Extract BusyBox

```bash
mkdir ~/qemu-rootfs
cd ~/qemu-rootfs

wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
```

### Step 3: Configure BusyBox

```bash
make defconfig
make menuconfig
```

In the menuconfig menu, enable:
```
Settings
  └── Build static binary (no shared libs)
```

This creates a static binary with no external dependencies.

### Step 4: Build BusyBox

```bash
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
make CROSS_COMPILE=aarch64-linux-gnu- CONFIG_PREFIX=../qemu-rootfs/rootfs install
```

### Step 5: Create Required Directories

```bash
cd ../qemu-rootfs/rootfs
mkdir -p proc sys dev tmp
mkdir -p etc/init.d
```

These are essential directories in any Linux filesystem:
- `proc` — Virtual filesystem exposing kernel information
- `sys` — Sysfs for device and driver information
- `dev` — Device files
- `tmp` — Temporary files
- `etc/init.d` — Initialization scripts

### Step 6: Create Init Script

Create the startup script:

```bash
vi etc/init.d/rcS
```

Add the following content:

```sh
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev

echo "BusyBox RootFS Booted Successfully"

/bin/sh
```

Make it executable:

```bash
chmod +x etc/init.d/rcS
```

### Step 7: Create Init Symlink

The kernel expects an `init` binary at the root. Create a symlink:

```bash
cd ~/qemu-rootfs/rootfs
ln -s /bin/busybox init
ls -l init
```

### Step 8: Create RootFS Image File

Create a filesystem image:

```bash
cd ~/qemu-rootfs
dd if=/dev/zero of=rootfs.ext4 bs=1M count=128
mkdir mnt
sudo mount rootfs.ext4 mnt
sudo cp -r rootfs/* mnt/
sudo umount mnt
```

This creates a 128MB ext4 filesystem populated with all the BusyBox binaries and directories.

### Step 9: Boot BusyBox

```bash
qemu-system-aarch64 -M virt \ 
-cpu cortex-a57 \
-m 1024 \
-kernel kernel \
-drive file=rootfs.ext4,format=raw,if=virtio \
-device virtio-9p-device,fsdev=fsdev0,mount_tag=hostshare \
-append "root=/dev/vda rw console=ttyAMA0" \
-nographic
```
---

## QEMU Parameters Explained

| Option            | Description                                           |
| ----------------- | ----------------------------------------------------- |
| `-M versatilepb`  | Virtual board type (ARM Versatile)                    |
| `-M virt`         | Generic virtual ARM/ARM64 board                       |
| `-cpu cortex-a57` | Virtual CPU type (ARM Cortex-A57)                     |
| `-m 1024`         | RAM size in MB                                        |
| `-kernel`         | Path to Linux kernel image                            |
| `-drive`          | Virtual block device (filesystem)                     |
| `-append`         | Kernel boot arguments                                 |
| `-nographic`      | Terminal-only mode (no GUI)                           |
| `-net nic`        | Virtual network interface                             |
| `-net user`       | User-mode networking (NAT)                            |
| `-s`              | Enable GDB server on port 1234                        |
| `-S`              | Freeze CPU at startup                                 |

---

## Practical Folder Sharing

### Share a Folder Between Host and Guest

This is very useful for transferring files or sharing development code.

#### On the Host PC

Create a shared directory:

```bash
mkdir ~/qemu-shared
echo "Hello QEMU" > ~/qemu-shared/test.txt
```

#### QEMU Command with 9P Filesystem Sharing

```bash
qemu-system-aarch64 \
-M virt \
-cpu cortex-a57 \
-m 1024 \
-kernel kernel \
-drive file=rootfs.ext4,format=raw,if=virtio \
-fsdev local,id=fsdev0,path=/home/raj-pc/qemu-shared,security_model=none \
-device virtio-9p-device,fsdev=fsdev0,mount_tag=hostshare \
-append "root=/dev/vda rw console=ttyAMA0" \
-nographic
```

#### Inside Guest (QEMU Shell)

```bash
mkdir /mnt/host
mount -t 9p -o trans=virtio hostshare /mnt/host
ls /mnt/host
cat /mnt/host/test.txt
```

This mounts the host directory inside the guest, allowing bidirectional file sharing.

---

## Kernel Debugging with GDB

One of the most powerful features of QEMU is integrated GDB debugging. You can step through kernel code, set breakpoints, and inspect memory.

### Step 1: Start QEMU in Debug Mode

```bash
qemu-system-aarch64 \
-M virt \
-cpu cortex-a57 \
-m 1024 \
-kernel kernel \
-S -s \
-nographic
```

The flags mean:
- `-S` — Stop CPU at startup (wait for debugger)
- `-s` — Enable GDB server on port 1234

### Step 2: Connect GDB (In Another Terminal)

```bash
gdb vmlinux
```

Inside the GDB prompt:

```gdb
target remote :1234
b start_kernel
c
```

Commands explained:
- `target remote :1234` — Connect to QEMU's GDB server
- `b start_kernel` — Set breakpoint at kernel start function
- `c` — Continue execution until breakpoint

### Step 3: Debug Kernel Boot

Once the breakpoint is hit, you can:

```gdb
info registers        # View CPU registers
bt                    # View call stack
s                     # Step into function
n                     # Step over function
p variable_name       # Print variable value
b filename.c:100      # Set breakpoint at specific line
```

## Exit QEMU

To quit QEMU:

```
Ctrl + Shift + A, then X
```

Or from another terminal:

```bash
killall qemu-system-aarch64
```

---

##  Useful QEMU Boards

| Board       | Architecture      | Use Case                    |
| ----------- | ----------------- | --------------------------- |
| versatilepb | ARM 32-bit        | Classic ARM testing         |
| vexpress-a9 | ARM Cortex-A9     | Multi-core ARM testing      |
| virt        | ARM/ARM64 generic | Generic embedded systems    |
| raspi2b     | ARM Cortex-A7     | Raspberry Pi emulation      |
| pc          | x86               | Standard PC emulation       |
| q35         | x86-64            | Modern x86 emulation        |

---