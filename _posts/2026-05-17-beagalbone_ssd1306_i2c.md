---
title: BeagleBone SSD1306-I2C
date: 2026-05-17
categories: [Projects]
tags: [projects, ssd1306, i2c]
---

# SSD1306 I2C Linux Device Driver
This project implements a <u>Linux kernel driver for the SSD1306 OLED display using the I2C bus</u> and a small drawing helper library with fonts, primitives and buffering. It focuses on a tiny, structured user API so user-space programs can request drawing operations using `ioctl` without handling low-level bus transactions.

## 1. I2C Protocol Overview

I2C (Inter-Integrated Circuit) is a synchronous, multi-master, multi-slave serial bus.

Key concepts:

- Two bidirectional lines: `SDA` (data) and `SCL` (clock).
- Each slave device has a 7-bit or 10-bit address.
- A master initiates communication with a `START` condition and finishes with a `STOP` condition.
- Each transferred byte is followed by an acknowledgment bit (`ACK` or `NACK`).
- I2C supports read and write transfers on the same bus.

### Basic I2C transaction

1. Master sends `START`.
2. Master sends the 7-bit address plus a read/write bit.
3. Slave responds with `ACK`.
4. Master sends or receives data bytes.
5. Each byte is ACKed by the receiver.
6. Master sends `STOP`.

### SSD1306 I2C specifics

The SSD1306 OLED controller can be addressed with `0x3C` or `0x3D` depending on wiring. In this project, the device tree uses `0x3C` for the addressing mode.
The SSD1306 expects a 

- `0x00` indicates command mode.
- `0x40` indicates data mode.

For example:

- `0x00 0xAE` writes the `DISPLAY_OFF` command.
- `0x40 <pixel data bytes...>` writes framebuffer data.

## 2. Linux Driver Code

The main kernel driver is implemented in `main.c`<br>
[Github Code ](https://github.com/rajmahto9876/SSD1306-BeagleBone-device-driver.git)

### <u>Driver registration:</u>
The driver registers as an `i2c_driver`:
```c
/*--------------------------------------------------------------
	Device Tree Matching ( By .compatible) -> NEW
----------------------------------------------------------------*/
const struct of_device_id match_table_id[] = 
{
	{.name = "ssd1306", .compatible = "raj,ssd1306", .data = (void*)0x0987654321},
	{}/* Null Termination */
};

MODULE_DEVICE_TABLE(of, match_table_id);

/*------------------------------------------------
	Name Matching ( .name) -> LEGACY
---------------------------------------------------*/
const struct i2c_device_id ssd1306_ids[] = 
{
	{.name = "ssd1306", .driver_data = 0x12349876},
	{ }, /* Null Termination */
};

MODULE_DEVICE_TABLE(i2c, ssd1306_ids);

/*------------------------------------------------
			INITILISATON PROBE
---------------------------------------------------*/
static struct i2c_driver ssd1306_i2c_driver = 
{
	.driver = {
		.name	= "ssd1306",
		.of_match_table = match_table_id,
	},
	.probe		= ssd1306_probe,
	.remove		= ssd1306_remove,
	.command 	= ioctl_cmds,
	.id_table	= ssd1306_ids,
};

module_i2c_driver(ssd1306_i2c_driver);
```

- `ssd1306_i2c_driver.driver.name = "ssd1306"`
- `probe = ssd1306_probe`
- `remove = ssd1306_remove`
- `id_table` supports legacy name matching.
- `of_match_table` supports device tree binding.

This allows the kernel to bind the driver to an I2C device with compatible device tree properties.

###  <u>Probe and remove: </u>

`ssd1306_probe()` is called when the driver is bound to the SSD1306 device.

It performs three main tasks:

1. store the `i2c_client` pointer in `ssd1306_dev.ssd_client`
2. initialize a character device node with `init_device()`
3. initialize the display hardware with `initilse_ssd1306()`

`ssd1306_remove()` cleans up the char device and unregisters resources.

###  <u>Character device interface: </u>

The driver creates a character device named `/dev/ssd1306_device`.

The file operations are declared in `main.c`.

Currently the stubs are:

- `fops_open`
- `fops_release`
- `fops_read`
- `fops_write`
- `fops_llseek`
- `fops_unlocked_ioctl`

These are implemented as placeholders that log calls, but they show how a user-space interface is intended to be built around the driver.

### <u>I2C transfers: </u>

The driver sends commands and data using kernel I2C helpers:

- `i2c_master_send(ssd1306_dev.ssd_client, buf, len)`

The repo exposes three key helper functions:

- `ssd_write_cmd(int cmd)` writes a single command byte with `0x00` as the control prefix.
- `ssd1306_write_data(int data)` writes a single data byte with `0x40` as the control prefix.
- `ssd1306_write_data_multi(char *data, int size)` writes a block of data.

These functions are the low-level I2C bridge between the SSD1306 display and the display helper library.

## 3. SSD1306 Helper Layer

The helper layer lives in `ssd1306_helper.c` and `ssd1306_helper.h`.

It includes:

- Display buffer management.
- Graphic primitives.
- Text rendering functions.
- Initialization and clearing of the OLED.
- Definitions for SSD1306 commands and parameters.

`ssd1306_helper.h` defines the display address, commands, and control bytes used by the driver.

The helper layer is responsible for the OLED-specific command sequence, while `main.c` handles kernel/I2C integration.

## 4. Device Tree Overlay

The overlay file is `ssd1306.dts`.

```c
/dts-v1/;
/plugin/;

/ {

    compatible = "ti,am335x-bone-black";

    fragment@0 {

        target = <&i2c2>;

        __overlay__ {

            status = "okay";

            #address-cells = <1>;
            #size-cells = <0>;

            ssd1306@3c {

                compatible = "raj,ssd1306";

                reg = <0x3c>;
            };
        };
    };
};
```
It enables the I2C bus and defines an SSD1306 node:

- target: `&i2c2`
- status: `"okay"`
- `ssd1306@3c` with compatible string `"raj,ssd1306"`
- `reg = <0x3c>;`

This is the binding point used by the driver `of_match_table`:

- `.compatible = "raj,ssd1306"`

When loaded on a board that supports `i2c2`, the kernel can attach this driver to the SSD1306 device.

### <u>U-Boot Changes: </u>
Since the Device Tree Overlay is used the U-Boot need to be changed. The U-Boot is changed
to add the .dtbo on the boot time.

```shell
sudo cp ssd1306.dtbo /lib/firmware/

#uEnv.txt file changes.
sudo vi /boot/uEnv.txt
enable_uboot_overlays=1
uboot_overlay_addr4=/lib/firmware/ssd1306.dtbo
enable_uboot_cape_universal=0

#reboot, so that .dtbo changes take place.
sudo reboot

# check this after reboot to confirm the dtb overlay worked.
ls /sys/bus/i2c/devices/


Expected:
2-003c
i2c-2

#
```

## 5. Build and Deployment

The `Makefile` shows how to build the module for cross-compilation on ARM.

```bash
obj-m := ssd1306.o
ssd1306-objs = main.o ssd1306_helper.o
ARCH=arm
CROSS_COMPILE=arm-linux-gnueabihf-

#set KERN_DIR to linux source location (Beaglebone)
KERN_DIR = beagalbone/linux

#set KERN_DIR to linux source location (Linux Ubuntu)
HOST_KERN_DIR = /lib/modules/$(shell uname -r)/build/

ccflags-y := -DMODULE_DEBUG

all:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) modules

dtc:
	dtc -@ -I dts -O dtb -o ssd1306.dtbo ssd1306.dts

clean:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) clean
	rm *.dtbo

help:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) help

user_app:
	gcc user_app.c -o user_app.o

clean_user:
	rm user_app.o
```
Important targets:

- `make all` builds the kernel module using `ARCH=arm` and `CROSS_COMPILE=arm-linux-gnueabihf-`
- `make dtc` compiles the DT overlay into `ssd1306.dtbo`
- `make clean` removes built objects.


## 6. User-space Example


### IOCTL API and data model
------------------------
The driver accepts user-space requests via a single structured payload: `shape_type_t` (defined in `user_ioctl_interface.h`). The payload contains:

- `gui_type` — enum selecting the operation: `BLANK_CIRCLE`, `FILLED_CIRCLE`, `BLANK_RECTANGLE`, `FILLED_RECTANGLE`, `TEXT`, `LOGO`.
- `shape` — a union with one of:
  - `t_text` — contains `text_cursor_x`, `text_cursor_y`, `text_size`, `text_color`, `buf_size`, `buffer[64]`.
  - `t_rect` — contains rectangle coordinates, width/height and color.
  - `t_logo` — contains `logo_cursor_x`, `logo_cursor_y`, `logo_size`, `logo_color`, `logo_buf_size`, `logo_buf[MAX_LOGO_SIZE]`.

This keeps the API compact and explicit: user-space fills `shape_type_t` and sends it using `IOCTL_SET_VALUE`.

IOCTL identifiers (example client)
----------------------------------
The example user app defines the ioctl numbers as follows (see `user_app.c`):

```c
#define MY_IOC_MAGIC                's'
#define IOCTL_SET_VALUE             _IOW(MY_IOC_MAGIC, 1, shape_type_t)
#define IOCTL_GET_VALUE             _IOR(MY_IOC_MAGIC, 2, shape_type_t)
```

### Example: drawing a logo or text:
--------------------------------
The repository contains `user_app.c` which demonstrates two simple operations:

1) Prepare a `shape_type_t` with `gui_type = LOGO`, set cursor, color and copy a small bitmap into `logo_buf`, then call `ioctl(fd, IOCTL_SET_VALUE, &shape_type)`.

2) After a delay, prepare `shape_type_t` with `gui_type = TEXT`, fill `text.buffer` and send another `IOCTL_SET_VALUE` to render text.

Snippet (logo):

```c
shape_type_t shape;
memset(&shape, 0, sizeof(shape_type_t));
shape.gui_type = LOGO;
shape.shape.logo.logo_cursor_x = 10;
shape.shape.logo.logo_cursor_y = 0;
shape.shape.logo.logo_color = WHITE;
shape.shape.logo.logo_buf_size = sizeof(college_logo);
memcpy(shape.shape.logo.logo_buf, college_logo, sizeof(college_logo));
ioctl(fd, IOCTL_SET_VALUE, &shape);
```

Snippet (text):

```c
memset(&shape, 0, sizeof(shape_type_t));
shape.gui_type = TEXT;
shape.shape.text.text_size = 1;
shape.shape.text.text_cursor_x = 1;
shape.shape.text.text_cursor_y = 0;
shape.shape.text.text_color = WHITE;
shape.shape.text.buf_size = strlen(text_print);
memcpy(shape.shape.text.buffer, text_print, strlen(text_print));
ioctl(fd, IOCTL_SET_VALUE, &shape);
```

### Build & run (quick)
-------------------
The repository `Makefile` has targets for cross-compilation, host builds, DT compilation and small helpers. Example commands (run from repo root):

```bash
# Cross-compile for BeagleBone (default ARCH/CROSS_COMPILE in Makefile)
make all

# Compile device-tree overlay
make dtc

# Insert built module (adjust filename as needed)
sudo insmod ssd1306.ko

# Build example user app
make user_app
./user_app.o
```

## 7. The Code Guide

This example demonstrates how a Linux kernel I2C driver is structured:

- Use `i2c_driver` and `i2c_client` for device binding.
- Provide a `probe()` callback to initialize resources.
- Implement a character device for user-space access.
- Use `i2c_master_send()` to transfer bytes over I2C.
- Use device tree overlays for hardware binding.

