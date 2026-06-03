---
title: Character Driver
date: 2026-05-30
categories: [Linux-Driver-Development]
tags: [linux, char driver Project]
---

# Linux Character Device Driver

Linux device drivers are kernel modules that manage communication between the operating system and hardware devices. A **character device driver** is one of the most fundamental types of device drivers in Linux, handling devices that process data as a stream of characters—such as terminals, serial ports, and other byte-oriented devices.

---

## Character Device Driver Overview

The character device driver we're examining implements a simple **temp character device** with 512 bytes of virtual buffer memory. It demonstrates essential concepts like device registration, file operations, and kernel-user space communication.

### Key Components:

1. **Device Buffer**: A 512-byte virtual buffer to simulate device memory
2. **Device Number**: Dynamically allocated major and minor numbers
3. **Character Device Structure (cdev)**: Manages device registration
4. **Device Class**: Creates sysfs entries under `/sys/class/`
5. **File Operations**: Read, write, seek, and control operations

---

## Detailed Code Explanation

### 1. Headers and Macros

```c
#include<linux/module.h>       // Core module infrastructure
#include<linux/fs.h>           // File system operations
#include<linux/cdev.h>         // Character device structures
#include<linux/device.h>       // Device management
#include<linux/kdev_t.h>       // Device number types
#include<linux/uaccess.h>      // User space access functions

#define MAX_DEVICE_COUNT	7    // Support up to 7 device instances
#define DVC_MEM_SIZE    	512  // Virtual device buffer size
```

**Why these headers?**
- `module.h`: Provides module initialization/cleanup infrastructure
- `fs.h`: File system operations and file structures
- `cdev.h`: Character device registration functions
- `device.h`: Device creation and class management
- `uaccess.h`: Safe copy functions between kernel and user space

### 2. Global Variables

```c
char dvc_buf[DVC_MEM_SIZE];        // Pseudo device buffer
dev_t dvc_num;                     // Stores device number
struct cdev char_cdev;             // Character device structure
struct class *pchar_Class;         // Device class pointer
struct device *pDvc;               // Device pointer
```

These variables hold the driver state throughout its lifetime.

---

## File Operations Callbacks

### `char_lseek()` - File Positioning

```c
static loff_t char_lseek(struct file *pFops, loff_t offset, int whence)
```

This function handles the `lseek()` system call, allowing users to move the file pointer within the device buffer. It supports three positioning modes:

- **SEEK_SET**: Absolute positioning from the start of the buffer
- **SEEK_CUR**: Relative positioning from the current position
- **SEEK_END**: Positioning relative to the end of the buffer

**Key Features:**
- Validates that the new position stays within buffer boundaries (0 to DVC_MEM_SIZE)
- Returns `-EINVAL` for invalid positions
- Updates `pFops->f_pos` to track the current position
- Prints debug information using the `pr_info()` macro

**Example:**
```
User calls lseek(fd, 100, SEEK_SET)
→ Driver sets file position to byte 100
→ Next read/write operates from position 100
```

### `char_read()` - Reading from Device

```c
static ssize_t char_read(struct file *pFops, char __user *buff, 
                        size_t count, loff_t *pFpos)
```

This function handles the `read()` system call. It reads data from the kernel buffer and safely copies it to user space.

**Operation Steps:**

1. **Validate Count**: Ensures we don't read beyond the buffer size
   ```c
   if((*pFpos + count) > DVC_MEM_SIZE)
       count = DVC_MEM_SIZE - *pFpos;
   ```

2. **Copy to User Space**: Uses `copy_to_user()` for safe data transfer
   ```c
   if(copy_to_user(buff, &dvc_buf[*pFpos], count))
       return -EFAULT;  // Fault if copy fails
   ```

3. **Update File Position**: Advances the pointer
   ```c
   *pFpos = *pFpos + count;
   ```

4. **Return Bytes Read**: Returns the number of bytes actually read

**`copy_to_user()`**
- User space memory is not directly accessible from kernel mode on many architectures
- This function performs necessary page table lookups and copying with proper error handling
- Prevents crashes from accessing invalid user space addresses

### `char_write()` - Writing to Device

```c
static ssize_t char_write(struct file *pFops, const char __user *buff, 
                         size_t count, loff_t *pFpos)
```

This function handles the `write()` system call, performing the inverse operation of `read()`.

**Operation Steps:**

1. **Validate and Adjust Count**: Prevents buffer overflow
2. **Copy from User Space**: Uses `copy_from_user()` for safe data transfer
   ```c
   if(copy_from_user(&dvc_buf[*pFpos], buff, count))
       return -EFAULT;
   ```
3. **Update File Position**: Advances pointer after write
4. **Return Bytes Written**: Returns count of successfully written bytes

### `char_open()` - Device Open Handler

```c
static int char_open(struct inode *pInode, struct file *pFops)
```

Called when the user opens the device file. This implementation is minimal—it simply logs that the device was opened. In production drivers, you might:
- Initialize device hardware
- Check permissions
- Allocate driver-specific data structures

### `char_flush()` - Buffer Flushing

```c
static int char_flush(struct file *pFops, fl_owner_t id)
```

Called when a file descriptor is duplicated or closed. This driver doesn't perform any actual flushing but demonstrates the hook point.

### `char_release()` - Device Release Handler

```c
static int char_release(struct inode *pInode, struct file *pFops)
```

Called when the last file descriptor reference is closed. Used for cleanup operations like freeing resources or flushing pending data.

---

## File Operations Structure

```c
struct file_operations char_fops = {
     .llseek  = char_lseek,
     .read    = char_read,
     .write   = char_write,
     .open    = char_open,
     .flush   = char_flush,
     .release = char_release,
     .owner   = THIS_MODULE
};
```

This structure tells the kernel which functions to call for each operation. The `.owner = THIS_MODULE` prevents the module from being unloaded while its functions are in use.

---

## Driver Initialization: `char_driver_init()`

```c
static int __init char_driver_init(void)
```

This is the driver's entry point, called when the module is loaded. It performs five critical steps:

### Step 1: Allocate Device Number

```c
alloc_chrdev_region(&dvc_num, 0, MAX_DEVICE_COUNT, "chr_dev_sample");
pr_info("<Major>:<Minor> = %d:%d \n", MAJOR(dvc_num), MINOR(dvc_num));
```

- Requests a **dynamic** device number from the kernel
- Parameters: storage address, starting minor number (0), count (7), device name
- Returns a unique major number; minor numbers are assigned sequentially
- **Why dynamic allocation?** Avoids conflicts with other drivers using static assignments

### Step 2: Initialize Character Device

```c
cdev_init(&char_cdev, &char_fops);
```

Associates the file operations callbacks with the character device structure.

### Step 3: Register Character Device

```c
char_cdev.owner = THIS_MODULE;
cdev_add(&char_cdev, dvc_num, MAX_DEVICE_COUNT);
```

- Sets the module as the owner (prevents unloading while in use)
- Registers the device with the Virtual File System (VFS)
- Now the kernel routes file operations to our callbacks

### Step 4: Create Device Class

```c
pchar_Class = class_create("char_class");
```

Creates a class named "char_class" under `/sys/class/`. This organizes device types.

### Step 5: Create Device Node

```c
pDvc = device_create(pchar_Class, NULL, dvc_num, NULL, "char_dvd");
```

- Creates a device node `/sys/class/char_class/char_dvd`
- The udev daemon automatically creates `/dev/char_dvd` with proper permissions
- Users can now interact with the device!

---

## Driver Cleanup: `char_driver_clean()`

```c
static void __exit char_driver_clean(void)
```

This function reverses the initialization in the correct order (important!):

```c
device_destroy(pchar_Class, dvc_num);        // Remove device node
class_destroy(pchar_Class);                   // Remove class
cdev_del(&char_cdev);                         // Unregister character device
unregister_chrdev_region(dvc_num, MAX_DEVICE_COUNT); // Free device numbers
```

**Reverse order of Init**
- Prevents dangling references
- Ensures resources are freed properly
- Avoids kernel oops or hangs

---

## Module Declarations

```c
module_init(char_driver_init);
module_exit(char_driver_clean);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Raj Kumar Mahto");
MODULE_DESCRIPTION("A simple character driver module");
```

These macros register the initialization and cleanup functions with the kernel.

---

## Building the Driver

### Makefile Overview

```makefile
obj-m := main.o
ARCH=arm
CROSS_COMPILE=arm-linux-gnueabihf-

KERN_DIR = <dir>/linux
HOST_KERN_DIR = /lib/modules/$(shell uname -r)/build/
```

**Variables Explained:**
- `obj-m`: Module object files to build
- `ARCH`: Target architecture (ARM for BeagleBone)
- `CROSS_COMPILE`: Toolchain prefix for cross-compilation
- `KERN_DIR`: Path to ARM Linux kernel source (BeagleBone target)
- `HOST_KERN_DIR`: Path to host machine's kernel headers

### Build Commands

#### 1. **Build for BeagleBone (ARM)**

```bash
make all
```

**Command Details:**
```makefile
all:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) modules
```

- `ARCH=arm`: Compile for ARM architecture
- `CROSS_COMPILE=arm-linux-gnueabihf-`: Use ARM toolchain
- `-C $(KERN_DIR)`: Change to kernel source directory
- `M=$(PWD)`: Build external module in current directory
- `modules`: Build only the module (not entire kernel)

**Output:** `main.ko` - The compiled kernel module

#### 2. **Build for Host Machine (Linux Ubuntu)**

```bash
make host
```

**Command Details:**
```makefile
host:
	make -C $(HOST_KERN_DIR) M=$(PWD) modules
```

- Uses host machine's kernel headers
- No cross-compilation flags needed
- Useful for testing on development PC before deploying to BeagleBone

#### 3. **Clean Build Artifacts**

```bash
make clean
```

**Command Details:**
```makefile
clean:
	make -C $(HOST_KERN_DIR) M=$(PWD) clean
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) clean
```

- Removes compiled `.o` files, `.ko` files, and build artifacts
- Cleans both host and target builds
- Useful before rebuilding from scratch

---

## Complete Build and Load Workflow

### For BeagleBone Deployment:

```bash
# 1. Build the module for ARM
make all

# 2. Copy to BeagleBone
make copy

# 3. SSH into BeagleBone and load the module
ssh debian@192.168.6.2
insmod main.ko

# 4. Check kernel logs
dmesg | tail -20

# 5. Test the device
cat /dev/char_dvd
echo "Hello" > /dev/char_dvd

# 6. Unload the module
rmmod main
```

### For Host Machine Development:

```bash
# 1. Build for host
make host

# 2. Load the module (requires root)
sudo insmod main.ko

# 3. Check kernel logs
sudo dmesg | tail -20

# 4. Clean up
sudo rmmod main
make clean
```

---
## Resources

- Linux Kernel Documentation: `Documentation/driver-model/`
- Character Device Drivers: `drivers/char/` in kernel source
- Device Number Management: Major/minor number allocation

---

**Author:** Raj Kumar Mahto  
**License:** GPL  
**Target Platform:** BeagleBone Black (ARM) / Linux x86_64
