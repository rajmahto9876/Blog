---
title: BeagleBone MPU9250 - SPI
date: 2026-05-21
categories: [Projects]
tags: [projects, mpu9250, spi]
---

This repository contains a Linux kernel module that exposes an MPU9250 IMU over SPI as a character device. The driver hides low-level SPI register handling and provides a simple user-space API via `ioctl` and a well-defined `mpu9250_t` data structure.

## What this driver exposes

- A character device at `/dev/mpu9250_device`
- An `ioctl` command to request the latest sensor sample
- A kernel-side helper API for initialization, register read/write, and data collection
- A model for SPI-based sensor integration using device tree matching and `spi_driver`

## Hardware wiring (BeagleBone Black SPI1)

| MPU9250 Pin | BeagleBone Black Pin | Purpose                  |
|-------------|----------------------|--------------------------|
| VCC         | 3.3V                 | Power                    |
| GND         | GND                  | Ground                   |
| SCK         | P9_31                | SPI Clock                |
| MISO        | P9_29                | Master In / Slave Out    |
| MOSI        | P9_30                | Master Out / Slave In    |
| CS          | P9_28                | SPI Chip Select (CS0)    |
| INT         | P9_12                | Interrupt / Data Ready   |

> Warning: MPU9250 is a 3.3V device. Do not drive it with 5V signals without proper level shifting.

## SPI settings

- Mode: SPI mode 0 (`CPOL=0`, `CPHA=0`)
- Clock: configured in the device tree (`dts`) for 1 MHz
- Read flag: set MSB of register address to `1`
- Write flag: clear MSB of register address to `0`

### SPI transaction format

- Read: send `[reg | 0x80]`, then clock out dummy bytes to receive data
- Write: send `[reg & 0x7F, value(s)...]`
- Burst read: send the register address once and continue clocking to receive a block of consecutive registers

## Driver API

The driver creates a character device named `mpu9250_device` and exports the following user-space interface:

- Device path: `/dev/mpu9250_device`
- `ioctl` command: `MPU_GET_DATA`
- Data structure: `mpu9250_t`

### Github Code:
The main kernel driver is implemented in `main.c`<br>
[Github Code ](https://github.com/rajmahto9876/MPu9250-BeagleBone-device-driver)

## Kernel-side helper API

The driver exposes several helper functions in `mpu9250_helper.h` for use inside the module or other kernel code:

- `u8 mpu9250_Init(void)` - initialize MPU9250 registers and sensor state
- `void mpu9250_Read_Accel(mpu9250_t *DataStruct)` - read accelerometer data
- `void mpu9250_Read_Gyro(mpu9250_t *DataStruct)` - read gyroscope data
- `void mpu9250_Read_Temp(mpu9250_t *DataStruct)` - read temperature data
- `void mpu9250_Read_All(mpu9250_t *DataStruct)` - read accel, gyro, and temperature together
- `int mpu9250_read_block_data(int reg, u8 *data, u8 size)` - low-level SPI block read
- `int mpu9250_write_block_data(int reg, const u8 *data, u8 size)` - low-level SPI block write

### Driver registration and device tree support

The driver registers as an `spi_driver` with both device tree and legacy SPI ID matching:

- `of_match_table` uses `.compatible = "raj,mpu9250"`
- `spi_device_id` uses `.name = "mpu9250"`

This allows the module to bind to the sensor via device tree overlays or platform SPI configuration.

### Device Tree Overlay
The driver registers as an `spi_driver`:
```c
/*================================================================
						INIT CALLS
=================================================================*/
/*--------------------------------------------------------------
	Device Tree Matching ( By .compatible) -> NEW
----------------------------------------------------------------*/
const struct of_device_id match_table_id[] = 
{
	{.name = "mpu9250", .compatible = "raj,mpu9250", .data = (void*)0x123476589},
	{}/* Null Termination */
};

MODULE_DEVICE_TABLE(of, match_table_id);

/*------------------------------------------------
	Name Matching ( .name) -> LEGACY
---------------------------------------------------*/
const struct spi_device_id mpu9250_ids[] = 
{
	{.name = "mpu9250", .driver_data = 0x1476589A},
	{ }, /* Null Termination */
};

MODULE_DEVICE_TABLE(spi, mpu9250_ids);

/*------------------------------------------------
			INITILISATON PROBE
---------------------------------------------------*/
static struct spi_driver mpu9250_spi_driver = 
{
	.driver = {
		.name	= "mpu9250",
		.of_match_table = match_table_id,
	},
	.probe		= mpu9250_probe,
	.remove		= mpu9250_remove,
	.id_table	= mpu9250_ids,
};

module_spi_driver(mpu9250_spi_driver);
```

### 4. Device Tree Overlay

The overlay file is `mpu9250.dts`.

```c
/dts-v1/;
/plugin/;

/ {
    compatible = "ti,beaglebone", "ti,beaglebone-black";

    fragment@0
    {

        target = <&spi1>;

        __overlay__
        {

            status = "okay";

            pinctrl-names = "default";

            #address-cells = <1>;
            #size-cells = <0>;

            mpu9250@0
            {

                compatible = "raj,mpu9250";

                /*
                    | CS Number | Physical Pin |
                    | --------- | ------------ |
                    | CS0       | P9_28        |
                    | CS1       | P9_42        |
                */
                
                reg = <0>;  // means use chipselect zero

                spi-max-frequency = <1000000>;

                interrupt-parent = <&gpio1>;
                interrupts = <28 0>;
            };
        };
    };
};
```
It enables the spi bus and defines an SSD1306 node:

- target: `&spi1`
- status: `"okay"`
- `mpu9250@0` with compatible string `"raj,mpu9250"`
- `reg = <0x00>;`

This is the binding point used by the driver `of_match_table`:

- `.compatible = "raj,mpu9250"`

When loaded on a board that supports `spi1`, the kernel can attach this driver to the MPU9250 device.

## User-space Code
### `mpu9250_t` fields

The sensor payload returned by `MPU_GET_DATA` includes:

- `Accel_X_RAW`, `Accel_Y_RAW`, `Accel_Z_RAW`
- `Ax`, `Ay`, `Az`
- `Gyro_X_RAW`, `Gyro_Y_RAW`, `Gyro_Z_RAW`
- `Gx`, `Gy`, `Gz`
- `Temperature`

### User-space Interface

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include "mpu9250_user_interface.h"

#define MPU_IOCTL_MAGIC 'm'
#define MPU_GET_DATA _IOR(MPU_IOCTL_MAGIC, 1, mpu9250_t)
#define MPU_FILE_NAME "/dev/mpu9250_device"

int main(void)
{
    int fd = open(MPU_FILE_NAME, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    mpu9250_t data;
    if (ioctl(fd, MPU_GET_DATA, &data) == -1) {
        perror("ioctl");
        close(fd);
        return 1;
    }

    printf("Temp: %d\n", data.Temperature);
    close(fd);
    return 0;
}
```

## Build and deployment

- Kernel module sources: `main.c`, `mpu9250_helper.c`
- Interface headers: `mpu9250_helper.h`, `mpu9250_user_interface.h`
- User-space example: `user_app.c`


Use the repository `Makefile` to build the module and install it into the running kernel.

