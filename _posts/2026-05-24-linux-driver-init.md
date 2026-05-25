---
title: Init Project (First Module)
date: 2026-05-24
categories: [Linux-Driver-Development]
tags: [linux, Init Project]
---

# Hello-World Module 

## Introduction

This blog explains a simple Linux kernel module. The "Hello World" kernel module, demonstrate how to create, build, load, and unload a kernel module using `main.c` and a `Makefile`.

---

## Source Code: `main.c`

```c
#include <linux/module.h>

static int __init my_entry_func(void)
{
    pr_info("Hello World \n");
    return 0;
}

static void __exit my_exit_func(void)
{
    pr_info("Exit World \n");
}

module_init(my_entry_func);
module_exit(my_exit_func);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Raj Kumar Mahto");
MODULE_DESCRIPTION("A simple hello world kernel module");
```

### API Brief Explanation:

- `my_entry_func()` runs when the module is inserted.
- `pr_info("Hello World")` writes a message to the kernel log.
- `my_exit_func()` runs when the module is removed.
- `pr_info("Exit World")` writes a cleanup message.

### Key Concepts

- `__init` marks the initialization code for one-time use.
- `__exit` marks the cleanup code.
- `module_init()` registers the module's load function.
- `module_exit()` registers the module's unload function.
- `pr_info()` logs messages to `dmesg`.
- `MODULE_LICENSE("GPL")` tells the kernel the module is GPL-licensed.
- `MODULE_AUTHOR`, `MODULE_DESCRIPTION`, and `MODULE_INFO` provide metadata.

---

## Build Script: `Makefile`

```makefile
#obj-n:- Don;t compile the Module
#obj-y : Compile the Module and Link with the Kernel Image
#obj-m:- Compile the Module as dynamically lodable module.

obj-m := main.o
ARCH=arm
CROSS_COMPILE=arm-linux-gnueabihf-

#set KERN_DIR to linux source location (BeagleBone)
KERN_DIR = <home>/linux

#set KERN_DIR to linux source location (Linux Ubuntu)
HOST_KERN_DIR = /lib/modules/$(shell uname -r)/build/

all:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) modules

clean:
	make -C $(HOST_KERN_DIR) M=$(PWD) clean
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) clean

help:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) help

host:
	make -C $(HOST_KERN_DIR) M=$(PWD) modules
```

### Build Workflow

- `obj-m := main.o` instructs the kernel build system to compile `main.c` as a module.
- `ARCH=arm` sets the target architecture for BeagleBone.
- `CROSS_COMPILE=arm-linux-gnueabihf-` selects the cross-compiler prefix.
- `KERN_DIR` points to the BeagleBone Linux kernel source tree.
- `HOST_KERN_DIR` points to the host's installed kernel build directory.
- `make` uses the kernel build system to build external modules with `-C $(KERN_DIR) M=$(PWD) modules`.

### Useful Commands

- `make` — build the kernel module for the target board.
- `make clean` — remove generated module build files.
- `make help` — show kernel module build help.
- `make host` — build against the local host kernel source tree.

## Loading and Testing

After building the module, load it and verify using the kernel log:

```bash
# To Load:
sudo insmod main.ko
sudo dmesg | tail -n 5

# To Unload:
sudo rmmod main
sudo dmesg | tail -n 5
```

Expected output in `dmesg`:
```bash
#On load: 
- Hello World
# On unload: 
- Exit World
```
---