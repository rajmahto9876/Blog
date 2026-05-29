---
title: BeagleBone GPIO
date: 2026-05-16
categories: [Projects]
tags: [projects, gpio, toggle]
---

This GPIO kernel driver is designed for **BeagleBone Black** boards and demonstrates advanced Linux kernel programming concepts including:

- **Character Device Driver** implementation
- **GPIO Control** (LED on/off, Button input)
- **Interrupt Handling** with ISRs
- **Kernel Threads** for concurrent processing
- **Wait Queues** for process synchronization
- **ioctl** commands for user-space communication
- **Spinlocks** for thread-safe operations

The driver creates a character device (`/dev/gpio_device`) that allows user-space applications to:
- Toggle LEDs on/off using ioctl commands
- Read button states
- Handle interrupts asynchronously using kernel threads

---

## Architecture & Design

```
┌─────────────────────────────────────────────────────┐
│          User Application (user_app.c)              │
│     - Opens /dev/gpio_device                        │
│     - Sends ioctl commands (LED_ON, LED_OFF)        │
└──────────────────┬──────────────────────────────────┘
                   │ (ioctl system call)
┌──────────────────▼──────────────────────────────────┐
│      Character Device Driver (main.c)               │
│  ┌─────────────────────────────────────────────┐    │
│  │  File Operations Layer                      │    │
│  │  - open() / release()                       │    │
│  │  - read() / write()                         │    │
│  │  - unlocked_ioctl()                         │    │
│  │  - llseek()                                 │    │
│  └─────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────┐    │
│  │  Hardware Abstraction Layer                 │    │
│  │  - GPIO Kernel API                          │    │
│  │  - Interrupt Management                     │    │
│  │  - Kernel Thread Pool                       │    │
│  └─────────────────────────────────────────────┘    │
└──────────────────┬──────────────────────────────────┘
                   │ (System calls)
┌──────────────────▼──────────────────────────────────┐
│          Linux Kernel Core                          │
│  - VFS (Virtual File System)                        │
│  - IRQ Management                                   │
│  - Process Scheduler                                │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│          Hardware (BeagleBone Black)                │
│  - GPIO Pin 60 (LED)                                │
│  - GPIO Pin 48 (Button)                             │
│  - Interrupt Controller                             │
└─────────────────────────────────────────────────────┘
```

---

## Included Headers

### Linux Kernel Module Headers

```c
#include<linux/module.h>      // Required for kernel module structure
#include<linux/fs.h>          // File system operations and structures
#include<linux/cdev.h>        // Character device operations
#include<linux/device.h>      // Device class and device creation
#include<linux/kdev_t.h>      // Device number (major/minor) handling
#include<linux/uaccess.h>     // User-space/Kernel-space data transfer
#include<linux/gpio.h>        // GPIO kernel API functions
#include<linux/gpio/machine.h>// GPIO machine configuration
#include<linux/interrupt.h>   // Interrupt handling functions
#include<linux/errno.h>       // Error number definitions
#include<linux/kthread.h>     // Kernel thread creation and management
#include<linux/delay.h>       // Delay functions (msleep, udelay)
#include<linux/wait.h>        // Wait queue for synchronization
#include<linux/poll.h>        // Poll mechanism for multiplexing
#include<linux/atomic.h>      // Atomic operations for thread safety
```

#### Purpose of Each Header:

| Header | Purpose |
|--------|---------|
| `module.h` | Provides `module_init()`, `module_exit()`, and MODULE_* macros |
| `fs.h` | Defines `file_operations`, `inode`, `file` structures |
| `cdev.h` | Character device structure and operations |
| `device.h` | Device class and device creation functions |
| `kdev_t.h` | Functions to work with device numbers (MAJOR, MINOR macros) |
| `uaccess.h` | `copy_to_user()` and `copy_from_user()` for safe data transfer |
| `gpio.h` | `gpio_request()`, `gpio_free()`, `gpio_get_value()`, `gpio_set_value()` |
| `interrupt.h` | `request_irq()`, `free_irq()`, `IRQ flags` |
| `kthread.h` | Kernel thread creation (`kthread_create()`, `kthread_stop()`) |
| `wait.h` | Wait queue initialization and macros |
| `atomic.h` | Atomic variable operations (`atomic_set()`, `atomic_read()`) |

---

## Macro Definitions

### 1. **Debug Output Macro**

```c
#undef pr_fmt
#define pr_fmt(fmt) "%s : "fmt,__func__
```

- **Purpose**: Formats all `pr_info()`, `pr_err()` messages to include function name
- **Usage**: When you call `pr_info("message")`, it prints `"function_name : message"`
- **Example**: `pr_info("LED ON")` prints `"gpio_driver_init : LED ON"`

### 2. **Buffer Configuration**

```c
#define MAX_KERNEL_BUFFER_SIZE 1024
```

- **Purpose**: Defines maximum size of kernel buffer for read/write operations
- **Usage**: Prevents buffer overflow when reading/writing data from user-space
- **Significance**: 1KB is sufficient for GPIO operations

### 3. **ioctl Magic Number & Commands**

```c
#define GPIO_IOCTL_MAGIC 'g'

#define GPIO_LED_ON   _IO(GPIO_IOCTL_MAGIC, 1)
#define GPIO_LED_OFF  _IO(GPIO_IOCTL_MAGIC, 2)
#define GPIO_GET_BTN  _IOR(GPIO_IOCTL_MAGIC, 3, int)
```

- **GPIO_IOCTL_MAGIC**: Unique identifier ('g') to prevent ioctl command conflicts
- **_IO()**: Command without data transfer
- **_IOR()**: Command with read access (kernel reads from user-space)
- **GPIO_LED_ON / OFF**: Commands to control LED state
- **GPIO_GET_BTN**: Command to read button state

#### GPIO Calculation

```
GPIO Pin Address Calculation:
Pin Number = (Bank * 32) + Pin_in_Bank
Example: GPIO Pin 60 = Bank 1, Pin 28
  60 = (1 * 32) + 28
```

### 4. **GPIO Pin Definitions**

```c
#define LED_GPIO_1    60        // GPIO Bank 1, Pin 28
#define BUTTON_GPIO_1 48        // GPIO Bank 1, Pin 16
```

- **GPIO 60**: Configured as OUTPUT for LED control
- **GPIO 48**: Configured as INPUT with interrupt for button

### 5. **GPIO State Definitions**

```c
#define HIGH 1
#define LOW  0
```

- **HIGH**: GPIO output level = 3.3V (LED ON)
- **LOW**: GPIO output level = 0V (LED OFF)

---

## Global Variables & Data Structures

### Device Management Variables

```c
static struct class *gpio_class = NULL;      // Device class (/sys/class/gpio_class/)
static struct device *gpio_dvc = NULL;       // Device entry (/dev/gpio_device)
static dev_t gpio_dvcNum = 0;               // Device number (major & minor)
static int gpio_baseMinor = 0;              // Base minor number (starts at 0)
static int gpio_devCount = 1;               // Number of devices (single device)
static struct cdev gpio_cdev;               // Character device structure
```

**Purpose**: These variables hold kernel structures for character device registration and management.

### Synchronization Variables

```c
static DEFINE_SPINLOCK(my_lock);            // Spinlock for thread-safe operations
static volatile int irq_args = 0xAA55;      // Private data for ISR
static volatile int thread_args = 0;        // Shared counter for threads
```

**Purpose**:
- **Spinlock**: Prevents race conditions when multiple threads access `thread_args`
- **irq_args**: Passed to ISR, value 0xAA55 used as identifier
- **thread_args**: Counter incremented by slave threads (protected by spinlock)

### Buffer Management

```c
static char kernel_buffer[MAX_KERNEL_BUFFER_SIZE];  // 1KB kernel buffer
```

**Purpose**: Stores data read from/written to user-space safely within kernel space.

### Naming Constants

```c
static char *class_name = "gpio_class";
static char *chrdev_name = "gpio_dvc_num";
static char *device_name = "gpio_device";
```

**Purpose**: Names used for device class and device file creation:
- Device appears as `/dev/gpio_device`
- Class appears in `/sys/class/gpio_class/`

### Kernel Thread Management

```c
static struct task_struct *kthread_main_handle = NULL;
static struct task_struct *kthread_slave_handle_1 = NULL;
static struct task_struct *kthread_slave_handle_2 = NULL;
```

**Purpose**: Handles to three kernel threads:
- **Main thread**: Waits for button interrupt, spawns slave threads
- **Slave thread 1 & 2**: Increment counter concurrently (demonstrates synchronization)

### Wait Queue & Event Flag

```c
static wait_queue_head_t gpio_wq;           // Wait queue for process synchronization
static atomic_t event_flag = ATOMIC_INIT(0); // Atomic flag (interrupt occurred?)
```

**Purpose**:
- **Wait Queue**: Suspends main thread until button interrupt occurs
- **Event Flag**: Atomic variable indicating interrupt has occurred (thread-safe)

---

## Interrupt Service Routine (ISR)

### ISR Implementation

```c
static irqreturn_t button_isr(int irq, void *dev_id)
{
    int button_state = 0;
    button_state = gpio_get_value(BUTTON_GPIO_1);
    
    pr_info("Button interrupt: %d\n", button_state);
    atomic_set(&event_flag, 1);
    wake_up_interruptible(&gpio_wq);

    return IRQ_HANDLED;
}
```

### Function Breakdown

| Operation | Purpose |
|-----------|---------|
| `gpio_get_value(BUTTON_GPIO_1)` | Read current button pin state (0 or 1) |
| `atomic_set(&event_flag, 1)` | Set event flag to indicate interrupt occurred |
| `wake_up_interruptible(&gpio_wq)` | Wake main thread waiting on `gpio_wq` |
| `return IRQ_HANDLED` | Tell kernel interrupt was successfully handled |

### ISR Execution Flow

```
Hardware Interrupt
        │
        ▼
┌──────────────────────┐
│  button_isr() calls  │
└──────────────────────┘
        │
        ├─→ Read button state
        │
        ├─→ Set event flag = 1
        │
        ├─→ Wake up waiting thread
        │
        └─→ Return IRQ_HANDLED
```

### Key Characteristics

- **Atomic Operations**: Uses `atomic_set()` because ISR can interrupt any kernel code
- **Minimal Work**: ISR does minimal work; main thread handles heavy lifting
- **Non-blocking**: Returns immediately without waiting

---

## Kernel Threads

The driver implements a thread pool with one main thread and two dynamically created slave threads.

### Slave Thread Function (slaveThreadfn_1 & slaveThreadfn_2)

```c
static int slaveThreadfn_1(void *args)
{
    while(!kthread_should_stop())
    {
        spin_lock(&my_lock);
        *((int *)args) = *((int *)args) + 1;
        // pr_info("Slaves_1 -> [%d]\n", *((int *)args));
        spin_unlock(&my_lock);
        msleep(10);
    }
    pr_info("Exit slaveThreadfn_1\n");
    return 0;
}
```

#### What It Does:

1. **Loop**: Continues until `kthread_should_stop()` is called
2. **Lock**: Acquires spinlock to prevent race conditions
3. **Increment**: Increments shared counter (`thread_args`)
4. **Unlock**: Releases spinlock
5. **Sleep**: Sleeps 10ms before next iteration
6. **Exit**: Logs exit message when stopped

**Purpose**: Demonstrates concurrent access to shared data with proper synchronization.

### Main Thread Function (mainThreadfn_1)

```c
static int mainThreadfn_1(void *args)
{
    int temp = 0;
    int count = 0;
    while(!kthread_should_stop())
    {
        // Create slave threads if they don't exist
        if(kthread_slave_handle_1 == NULL || kthread_slave_handle_2 == NULL)
        {
            pr_info("Waking up Slaves for Printing \n");
            
            if(kthread_slave_handle_1 == NULL)
            {
                kthread_slave_handle_1 = kthread_create(slaveThreadfn_1, 
                                                       (void *)&thread_args, 
                                                       "slaveThreadfn_1");
                if(IS_ERR(kthread_slave_handle_1))
                {
                    pr_err("Can't Create slaveThreadfn_1\n");
                }
                wake_up_process(kthread_slave_handle_1);
            }

            if(kthread_slave_handle_2 == NULL)
            {
                kthread_slave_handle_2 = kthread_create(slaveThreadfn_2, 
                                                       (void*)&thread_args, 
                                                       "slaveThreadfn_2");
                if(IS_ERR(kthread_slave_handle_2))
                {
                    pr_err("Can't Create slaveThreadfn_2\n");
                }
                wake_up_process(kthread_slave_handle_2);
            }
        }

        // Wait for button interrupt
        pr_info("gpio_read: waiting for event: event Occurred->[%d]\n", count);
        wait_event_interruptible(gpio_wq, atomic_read(&event_flag));
        atomic_set(&event_flag, 0);
        count++;

        // Read and log button state
        temp = gpio_get_value(BUTTON_GPIO_1);
        pr_info("gpio_read: returning value %d\n", temp);
        msleep(10);
    }
    return 0;
}
```

#### Execution Flow:

1. **Lazy Thread Creation**: Slave threads created on-demand when first interrupt occurs
2. **Error Checking**: Uses `IS_ERR()` to check for thread creation failures
3. **Process Wake**: `wake_up_process()` makes thread ready to run
4. **Wait for Interrupt**: Blocks until button interrupt (ISR sets event flag)
5. **Read State**: Gets button state after interrupt
6. **Counter**: Increments count with each interrupt

### Thread Life Cycle

```
Module Init
    │
    ├─→ Create Main Thread
    │       │
    │       └─→ Starts waiting for interrupt
    │
Button Press (Interrupt)
    │
    ├─→ ISR sets event flag
    │
    ├─→ Main thread wakes up
    │
    ├─→ Create Slave Threads (first time)
    │
    ├─→ Slave threads start incrementing counter
    │
    └─→ Main thread waits again

Module Cleanup
    │
    ├─→ Stop main thread
    │
    ├─→ Stop slave thread 1
    │
    └─→ Stop slave thread 2
```

### Thread Safety Considerations

```c
spin_lock(&my_lock);        // Acquire lock (disables preemption)
*((int *)args)++;           // Protected critical section
spin_unlock(&my_lock);      // Release lock (re-enables preemption)
```

**Why Spinlock?**
- **Kernel context**: Spinlock suitable for very short critical sections
- **Preemption**: Disables preemption to prevent context switch during lock

---

## File Operations

The driver implements standard file operations to expose GPIO functionality to user-space.

### File Operations Structure

```c
static const struct file_operations gpio_fops = 
{
    .open           = gpio_open,
    .release        = gpio_release,
    .unlocked_ioctl = gpio_unlocked_ioctl,
    .owner          = THIS_MODULE,
    .llseek         = gpio_llseek,
    .write          = gpio_write,
    .read           = gpio_read,
};
```

### 1. Open Operation

```c
int gpio_open(struct inode *pInode, struct file *pFile)
{
    pr_info("In gpio_open\n");
    return 0;
}
```

- **When Called**: When user opens `/dev/gpio_device`
- **Purpose**: Could perform initialization, permission checks, resource allocation
- **Current Implementation**: Minimal (just logging)

### 2. Release (Close) Operation

```c
int gpio_release(struct inode *pInode, struct file *pFile)
{
    pr_info("In gpio_release\n");
    return 0;
}
```

- **When Called**: When user closes file descriptor
- **Purpose**: Clean up resources, finalize operations
- **Current Implementation**: Minimal (just logging)

### 3. Read Operation

```c
ssize_t gpio_read(struct file *pFile, char __user *uBuff, 
                  size_t count, loff_t *pPos)
{
    pr_info("pcd_read requested %zu bytes\n", count);
    pr_info("current file pos = %lld \n", *pPos);

    /* 1. Adjust count if it exceeds buffer bounds */
    if((*pPos + count) > MAX_KERNEL_BUFFER_SIZE)
    {
        count = MAX_KERNEL_BUFFER_SIZE - *pPos;
    }

    /* 2. Copy data to user buffer */
    if(copy_to_user(uBuff, &kernel_buffer[*pPos], count))
    {
        pr_err("%s copy_to_user error \n", __func__);
        return -EFAULT;
    }
    
    /* 3. Update file position */
    *pPos = *pPos + count;

    pr_info("no of bytes successfully read %zu \n", count);
    pr_info("Update file pos %lld \n", *pPos);
    return count;
}
```

#### Read Operation Breakdown:

| Step | Action | Purpose |
|------|--------|---------|
| 1 | Validate count | Prevent reading beyond buffer |
| 2 | copy_to_user() | Safely copy data from kernel to user-space |
| 3 | Update file pointer | Track position for sequential reads |
| Return | Bytes read | Tells user-space how much data was read |

**Important Security Functions**:
- `copy_to_user()`: Prevents direct memory access from user-space; validates pointers
- Returns `-EFAULT` if user buffer is invalid

### 4. Write Operation

```c
ssize_t gpio_write(struct file *pFile, const char __user *uBuff, 
                   size_t count, loff_t *pPos)
{
    pr_info("pcd_write requested %zu bytes\n", count);
    pr_info("current file pos = %lld \n", *pPos);

    /* 1. Adjust count if it exceeds buffer bounds */
    if((*pPos + count) > MAX_KERNEL_BUFFER_SIZE)
    {
        count = MAX_KERNEL_BUFFER_SIZE - *pPos;
    }

    if(!count)
    {
        return -ENOMEM;
    }

    /* 2. Copy data from user buffer */
    if(copy_from_user(&kernel_buffer[*pPos], uBuff, count))
    {
        pr_err("%s copy_from_user error \n", __func__);
        return -EFAULT;
    }

    /* 3. Update file position */
    *pPos = *pPos + count;

    pr_info("no of bytes successfully written %zu \n", count);
    pr_info("Update file pos %lld \n", *pPos);
    return count;  // Should return 0 on success, but returns count here
}
```

#### Write Operation Breakdown:

Similar to read, but:
- Uses `copy_from_user()` to safely copy FROM user-space TO kernel
- Stores data in `kernel_buffer`
- Returns `count` (note: convention is to return bytes written or 0)

### 5. Seek (llseek) Operation

```c
loff_t gpio_llseek(struct file *pFile, loff_t offset, int whence)
{
    loff_t temp;
    pr_info("lseek_called \n");
    pr_info("Current File Pos %lld \n", pFile->f_pos);
    
    switch(whence)
    {
        case SEEK_SET:
        {
            if((offset > MAX_KERNEL_BUFFER_SIZE) || (offset < 0))
                return -EINVAL;
            pFile->f_pos = offset;
        }
        break;
        
        case SEEK_CUR:
        {
            temp = pFile->f_pos + offset;
            if((temp > MAX_KERNEL_BUFFER_SIZE) || (temp < 0))
                return -EINVAL;
            pFile->f_pos = temp;
        }
        break;

        case SEEK_END:
        {
            temp = MAX_KERNEL_BUFFER_SIZE + offset;
            if((temp > MAX_KERNEL_BUFFER_SIZE) || (temp < 0))
                return -EINVAL;
            pFile->f_pos = temp;
        }
        break;
        
        default:
            return -EINVAL;
    }

    pr_info("Update File pos %lld \n", pFile->f_pos);
    return 0;
}
```

#### Seek Operations:

| Mode | Operation | Example |
|------|-----------|---------|
| `SEEK_SET` | Absolute position | `lseek(fd, 100, SEEK_SET)` → Position = 100 |
| `SEEK_CUR` | Relative to current | `lseek(fd, 10, SEEK_CUR)` → Position += 10 |
| `SEEK_END` | Relative to end | `lseek(fd, -10, SEEK_END)` → Position = SIZE - 10 |

**Bounds Checking**: All operations validate that new position stays within `[0, MAX_KERNEL_BUFFER_SIZE]`

### 6. ioctl Operation

```c
static long gpio_unlocked_ioctl(struct file *pFile, unsigned int cmd, 
                                unsigned long arg)
{
    int value;

    pr_info("IN gpio_unlocked_ioctl with cmd [%02x]\n", cmd);
    switch (cmd)
    {
        case GPIO_LED_ON:
            gpio_set_value(LED_GPIO_1, 1);
            break;

        case GPIO_LED_OFF:
            gpio_set_value(LED_GPIO_1, 0);
            break;

        case GPIO_GET_BTN:
            value = gpio_get_value(BUTTON_GPIO_1);
            if (copy_to_user((int __user *)arg, &value, sizeof(int)))
                return -EFAULT;
            break;

        default:
            return -EINVAL;
    }

    return 0;
}
```

#### ioctl Commands Explained:

| Command | Operation | Details |
|---------|-----------|---------|
| `GPIO_LED_ON` | LED Control | Sets GPIO 60 HIGH (LED turns ON) |
| `GPIO_LED_OFF` | LED Control | Sets GPIO 60 LOW (LED turns OFF) |
| `GPIO_GET_BTN` | Button Read | Reads GPIO 48 state, copies to user buffer |

**Key Points**:
- `gpio_set_value()`: Sets GPIO output level
- `gpio_get_value()`: Reads GPIO input level
- `copy_to_user()`: Safely transfer button state to user-space
- Returns `-EFAULT` if user pointer is invalid
- Returns `-EINVAL` for unknown commands

---

## Driver Initialization

### init Function: `gpio_driver_init()`

```c
static int __init gpio_driver_init(void)
{
    int ret = 0;
    int irq_num = 0;
    pr_info("In GPIO Init\n");

    /* Step 1: Allocate device number */
    ret = alloc_chrdev_region(&gpio_dvcNum, gpio_baseMinor, gpio_devCount, chrdev_name);
    if(ret)
    {
        pr_err("Gpio Alloc char Error\n");
        return ret;
    }
    pr_info("Assigned Major [%d], Minor [%d] \n", MAJOR(gpio_dvcNum), MINOR(gpio_dvcNum));

    /* Step 2: Create device class */
    gpio_class = class_create(THIS_MODULE, class_name);
    if(IS_ERR(gpio_class))
    {
        pr_err("Gpio Class Error\n");
        goto unalloc_chrdev;
    }

    /* Step 3: Create device */
    gpio_dvc = device_create(gpio_class, NULL, gpio_dvcNum, NULL, device_name);
    if(IS_ERR(gpio_dvc))
    {
        pr_err("Gpio Device Error\n");
        goto class_destroy;
    }

    /* Step 4: Initialize and add cdev */
    cdev_init(&gpio_cdev, &gpio_fops);
    gpio_cdev.owner = THIS_MODULE;
    ret = cdev_add(&gpio_cdev, gpio_dvcNum, gpio_devCount);
    if(ret)
    {
        pr_err("Cdev Add Failed\n");
        goto cdev_del;
    }

    /* Step 5: Initialize synchronization */
    init_interrupts_sync();

    /* Step 6: Configure LED GPIO */
    ret = gpio_request(LED_GPIO_1, "LED_GPIO_1");
    if (ret) {
        pr_err("%s: failed to request GPIO_1: %d\n", __func__, ret);
        goto gpio_led_unassign;
    }
    gpio_direction_output(LED_GPIO_1, HIGH);

    /* Step 7: Configure Button GPIO */
    ret = gpio_request(BUTTON_GPIO_1, "button");
    if (ret)
    {
        pr_err("%s: failed to request BUTTON_GPIO_1: %d\n", __func__, ret);
        goto gpio_led_unassign;
    }
    gpio_direction_input(BUTTON_GPIO_1);

    /* Step 8: Setup IRQ for button */
    irq_num = gpio_to_irq(BUTTON_GPIO_1);
    ret = request_irq(irq_num, button_isr, 
                     IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING, 
                     "my_gpio_irq", (void*)&irq_args);
    if (ret) {
        pr_err("%s: failed to request BUTTON_GPIO_1: %d\n", __func__, ret);
        goto gpio_irq_unassign;
    }

    /* Step 9: Initialize kernel threads */
    ret = initilize_kthread();
    if(ret) 
    {
        pr_err("Initialize kthread Failed \n");
        goto deinitilise_kthread;
    }

    pr_info("GPIO Driver Initialized Successfully\n");
    return 0;

    /* Error handling with cascading cleanup */
deinitilise_kthread:
    free_irq(gpio_to_irq(BUTTON_GPIO_1), (void*)&irq_args);

gpio_irq_unassign:
    gpio_free(LED_GPIO_1);

gpio_led_unassign:
    cdev_del(&gpio_cdev);

cdev_del:
    device_destroy(gpio_class, gpio_dvcNum);

class_destroy:
    class_destroy(gpio_class);

unalloc_chrdev:
    unregister_chrdev_region(gpio_dvcNum, gpio_devCount);

    return ret;
}
```

### Initialization Steps Explained

#### **Step 1: Allocate Device Number**

```c
alloc_chrdev_region(&gpio_dvcNum, gpio_baseMinor, gpio_devCount, chrdev_name)
```

- **Purpose**: Register character device with kernel
- **Parameters**:
  - `gpio_dvcNum`: Pointer to store allocated device number
  - `gpio_baseMinor`: Starting minor number (0)
  - `gpio_devCount`: Number of devices (1)
  - `chrdev_name`: Name shown in `/proc/devices`
- **Returns**: 0 on success, negative error code on failure
- **Kernel Assigns**: Major number automatically; minor number starts at 0

#### **Step 2: Create Device Class**

```c
gpio_class = class_create(THIS_MODULE, class_name)
```

- **Purpose**: Create sysfs class for device grouping
- **Effect**: Creates `/sys/class/gpio_class/` directory
- **Usage**: Used by udev to create `/dev/` entries automatically

#### **Step 3: Create Device**

```c
gpio_dvc = device_create(gpio_class, NULL, gpio_dvcNum, NULL, device_name)
```

- **Purpose**: Register device under the class
- **Effect**: Creates `/dev/gpio_device` (if udev is running)
- **Also Creates**: `/sys/class/gpio_class/gpio_device/` sysfs entry

#### **Step 4: Initialize Character Device**

```c
cdev_init(&gpio_cdev, &gpio_fops);
cdev_add(&gpio_cdev, gpio_dvcNum, gpio_devCount);
```

- **cdev_init()**: Associates file operations (`gpio_fops`) with cdev structure
- **cdev_add()**: Registers cdev in kernel (kernel now calls our functions for file ops)

#### **Step 5: Initialize Wait Queue**

```c
init_interrupts_sync()  // Calls init_waitqueue_head(&gpio_wq)
```

- **Purpose**: Initialize wait queue used by main thread
- **Effect**: Thread can now wait on this queue

#### **Step 6: Configure LED GPIO**

```c
gpio_request(LED_GPIO_1, "LED_GPIO_1")      // Request exclusive use
gpio_direction_output(LED_GPIO_1, HIGH)     // Set as output, initial HIGH
```

- **gpio_request()**: Claims GPIO from kernel (prevents other drivers using it)
- **gpio_direction_output()**: Configures as output, initial state HIGH
- **Effect**: GPIO 60 ready for LED control

#### **Step 7: Configure Button GPIO**

```c
gpio_request(BUTTON_GPIO_1, "button")       // Request exclusive use
gpio_direction_input(BUTTON_GPIO_1)         // Set as input
```

- **gpio_request()**: Claims GPIO 48 exclusively
- **gpio_direction_input()**: Configures as input
- **Effect**: GPIO 48 ready to read button state

#### **Step 8: Setup Interrupt Handler**

```c
irq_num = gpio_to_irq(BUTTON_GPIO_1);
request_irq(irq_num, button_isr, 
           IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING, 
           "my_gpio_irq", (void*)&irq_args)
```

- **gpio_to_irq()**: Convert GPIO pin number to IRQ line number
- **request_irq()**: Register ISR (`button_isr`) for that IRQ line
- **Flags**:
  - `IRQF_TRIGGER_RISING`: Interrupt on rising edge (0→1)
  - `IRQF_TRIGGER_FALLING`: Interrupt on falling edge (1→0)
  - Combined: Interrupt on any edge change
- **Parameter**: `&irq_args` passed to ISR as `dev_id` (identifies this device)

#### **Step 9: Initialize Kernel Threads**

```c
ret = initilize_kthread();

static int initilize_kthread(void)
{
    kthread_main_handle = kthread_create(mainThreadfn_1, NULL, "mainThreadfn_1");
    if(IS_ERR(kthread_main_handle))
    {
        pr_err("Can't Create mainThreadfn_1\n");
        return -ENOMEM;
    }
    wake_up_process(kthread_main_handle);
    return 0;
}
```

- **kthread_create()**: Create kernel thread (but not running yet)
- **IS_ERR()**: Check for creation errors
- **wake_up_process()**: Make thread runnable
- **Effect**: Main thread starts executing and waits for button interrupt

### Error Handling Strategy

The initialization uses **goto-based error handling** for proper cleanup:

```
If Step N fails:
    ├─→ Free resources allocated in Step N
    ├─→ Free resources allocated in Step N-1
    ├─→ Free resources allocated in Step N-2
    └─→ ... and so on backwards
```

This ensures no resource leaks. For example:
- If GPIO IRQ request fails, button GPIO is freed but LED GPIO remains requested (wrong!)
- Actually, looking at code, `gpio_irq_unassign` should free BUTTON GPIO (BUG in original code)

**Proper cleanup order**:
```
1. Stop threads
2. Free IRQ
3. Free GPIOs (both)
4. Delete cdev
5. Destroy device
6. Destroy class
7. Unregister device region
```

---

## Driver Cleanup & Exit

### Exit Function: `gpio_driver_exit()`

```c
static void __exit gpio_driver_exit(void)
{
    pr_info("In GPIO Exit\n");

    /* 1. Stop kernel threads */
    deinitilize_kthread();

    /* 2. Free interrupt */
    free_irq(gpio_to_irq(BUTTON_GPIO_1), (void*)&irq_args);

    /* 3. Free GPIOs */
    gpio_free(LED_GPIO_1);

    /* 4. Delete character device */
    cdev_del(&gpio_cdev);

    /* 5. Destroy device */
    device_destroy(gpio_class, gpio_dvcNum);

    /* 6. Destroy device class */
    class_destroy(gpio_class);

    /* 7. Unregister device number */
    unregister_chrdev_region(gpio_dvcNum, gpio_devCount);

    pr_info("GPIO Driver Exited\n");
}
```

### deinitilize_kthread Function

```c
static int deinitilize_kthread(void)
{
    if(kthread_main_handle != NULL)
    {
        kthread_stop(kthread_main_handle);
    }

    if(kthread_slave_handle_1 != NULL)
    {
        kthread_stop(kthread_slave_handle_1);
    }

    if(kthread_slave_handle_2 != NULL)
    {
        kthread_stop(kthread_slave_handle_2);
    }

    return 0;
}
```

### Cleanup Operations Explained

| Operation | Purpose |
|-----------|---------|
| `kthread_stop()` | Stop and wait for thread termination |
| `free_irq()` | Unregister interrupt handler |
| `gpio_free()` | Release GPIO claim, allows other drivers to use |
| `cdev_del()` | Unregister character device (prevents new opens) |
| `device_destroy()` | Remove `/dev/gpio_device` entry |
| `class_destroy()` | Remove `/sys/class/gpio_class/` |
| `unregister_chrdev_region()` | Return device number to kernel |

### Module Init/Exit Macros

```c
module_init(gpio_driver_init);
module_exit(gpio_driver_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Raj Kumar");
MODULE_DESCRIPTION("Gpio Driver Beaglebone");
MODULE_INFO(board,"Beaglebone black REV A5");
```

- **module_init()**: Specifies initialization function
- **module_exit()**: Specifies cleanup function
- **MODULE_***: Metadata visible via `modinfo` command

---

## User Application

### user_app.c

```c
#include <stdio.h>
#include <fcntl.h>
#include <sys/ioctl.h>

#define GPIO_LED_ON  _IO('g', 1)

int main()
{
    int fd = open("/dev/gpio_device", O_RDWR);

    ioctl(fd, GPIO_LED_ON);

    return 0;
}
```

### Application Flow

```
1. Open Device:
   ├─→ open("/dev/gpio_device", O_RDWR)
   └─→ Returns file descriptor 'fd'

2. Send LED_ON Command:
   ├─→ ioctl(fd, GPIO_LED_ON)
   └─→ Kernel calls gpio_unlocked_ioctl()
       └─→ Sets GPIO 60 HIGH
           └─→ LED turns ON

3. Close and Exit:
   └─→ File implicitly closed when process exits
```
---

## Build System (Makefile)

### Makefile Analysis

```makefile
obj-m := main.o
bonegpio-objs += gpio_driver.o
ARCH=arm
CROSS_COMPILE=arm-linux-gnueabihf-

# Linux source directories
KERN_DIR = <host>/linux
HOST_KERN_DIR = /lib/modules/$(shell uname -r)/build/

# Compiler flags
ccflags-y := -DMODULE_DEBUG
```

### Build Targets

#### **all** (Compile for BeagleBone)

```bash
all:
    make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) modules
```

- **ARCH=arm**: Target ARM architecture
- **CROSS_COMPILE=arm-linux-gnueabihf-**: Use ARM compiler
- **-C $(KERN_DIR)**: Compile against BeagleBone kernel source
- **M=$(PWD)**: Compile module in current directory
- **Output**: `main.ko` (kernel object)

#### **host** (Compile for Host Machine)

```bash
host:
    make -C $(HOST_KERN_DIR) M=$(PWD) modules
```

- Compiles against current system's kernel
- Useful for testing on Linux PC

#### **clean** (Remove Build Artifacts)

```bash
clean:
    make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) clean

clean_host:
    make -C $(HOST_KERN_DIR) M=$(PWD) clean
```

#### **insert** (Load Module)

```bash
insert:
    sudo insmod main.ko
```

- Loads compiled `.ko` file into running kernel
- Calls `gpio_driver_init()` function

#### **remove** (Unload Module)

```bash
remove:
    sudo rmmod main.ko
```

- Unloads module from kernel
- Calls `gpio_driver_exit()` function

#### **copy** (Transfer to BeagleBone)

```bash
copy:
    scp *.ko debian@192.168.7.2:/home/debian/main.ko
```

- Copies `.ko` file to BeagleBone via SCP
- IP: 192.168.7.2, User: debian

#### **get_info** (Module Information)

```bash
get_info:
    file main.ko
    modinfo main.ko
```

- **file**: Shows binary format and architecture
- **modinfo**: Displays module metadata (author, license, etc.)

#### **show_logs** (View Kernel Messages)

```bash
show_logs:
    sudo dmesg -xT | tail -20
```

- **dmesg**: Kernel message buffer
- **-xT**: Human-readable timestamps
- **tail -20**: Show last 20 messages
- **Purpose**: View `pr_info()` and `pr_err()` output

#### **perm** (Set Device Permissions)

```bash
perm:
    sudo chmod 777 /dev/my_device
```

- Makes device readable/writable by all users
- Useful for testing without sudo

#### **preprocess** (Generate Preprocessed Code)

```bash
preprocess:
    make -C $(HOST_KERN_DIR) M=$(PWD) main.i
```

- Generates preprocessed C file (`main.i`)
- Shows expanded macros, includes
- Useful for debugging macro issues

---

## How It All Works Together

### Complete System Interaction

```
┌─────────────────────────────────────────────────────────────────┐
│                    SYSTEM INITIALIZATION                         │
└─────────────────────────────────────────────────────────────────┘

1. Build Phase:
   User runs: make all
   ├─→ Invokes ARM cross-compiler
   ├─→ Compiles main.c against BeagleBone kernel
   └─→ Produces main.ko

2. Load Module Phase:
   User runs: sudo insmod main.ko
   ├─→ Kernel loads main.ko into memory
   ├─→ Kernel calls gpio_driver_init()
   │   ├─→ Allocates device number
   │   ├─→ Creates /dev/gpio_device
   │   ├─→ Requests GPIO 60 (LED) as output
   │   ├─→ Requests GPIO 48 (Button) as input
   │   ├─→ Registers ISR for GPIO 48 interrupt
   │   └─→ Starts main kernel thread
   └─→ Driver ready for use

┌─────────────────────────────────────────────────────────────────┐
│                    BUTTON PRESS SCENARIO                         │
└─────────────────────────────────────────────────────────────────┘

1. User presses button:
   Hardware
   ├─→ GPIO 48 state changes (0→1 or 1→0)
   └─→ Generates interrupt signal

2. Interrupt Controller:
   ├─→ Routes interrupt to CPU
   └─→ Calls registered ISR (button_isr)

3. ISR Execution:
   button_isr()
   ├─→ Read button state: gpio_get_value(GPIO 48)
   ├─→ Set event_flag = 1 (atomic operation)
   ├─→ Wake main thread from wait queue
   └─→ Return IRQ_HANDLED

4. Main Thread Resumes:
   mainThreadfn_1()
   ├─→ Woken up by wait_event_interruptible()
   ├─→ Check if slave threads exist
   │   └─→ If not, create them
   ├─→ Read button state again
   ├─→ Log the state
   └─→ Go back to waiting

5. Slave Threads:
   slaveThreadfn_1() & slaveThreadfn_2()
   ├─→ Acquire spinlock
   ├─→ Increment thread_args counter
   ├─→ Release spinlock
   ├─→ Sleep 10ms
   └─→ Repeat

┌─────────────────────────────────────────────────────────────────┐
│                    USER APP CONTROLS LED                         │
└─────────────────────────────────────────────────────────────────┘

1. User application execution:
   user_app.c
   ├─→ fd = open("/dev/gpio_device", O_RDWR)
   │   └─→ Kernel calls gpio_open()
   ├─→ ioctl(fd, GPIO_LED_ON)
   │   └─→ Kernel calls gpio_unlocked_ioctl()
   │       ├─→ Detects GPIO_LED_ON command
   │       ├─→ Calls gpio_set_value(GPIO 60, 1)
   │       └─→ Returns 0 (success)
   └─→ return 0 (exit)

2. Hardware Result:
   GPIO 60 output
   ├─→ Voltage set to 3.3V (HIGH)
   └─→ LED turns ON (if connected)

┌─────────────────────────────────────────────────────────────────┐
│                    UNLOAD MODULE                                 │
└─────────────────────────────────────────────────────────────────┘

1. User runs: sudo rmmod main.ko
   Kernel calls gpio_driver_exit()
   ├─→ deinitilize_kthread()
   │   ├─→ kthread_stop(main thread)
   │   ├─→ kthread_stop(slave thread 1)
   │   └─→ kthread_stop(slave thread 2)
   ├─→ free_irq() - unregister interrupt handler
   ├─→ gpio_free(GPIO 60) - release LED GPIO
   ├─→ gpio_free(GPIO 48) - release Button GPIO
   ├─→ cdev_del() - unregister character device
   ├─→ device_destroy() - remove /dev/gpio_device
   ├─→ class_destroy() - remove /sys/class/gpio_class
   └─→ unregister_chrdev_region() - return device number

2. Result:
   ├─→ All resources freed
   ├─→ Module removed from memory
   └─→ System ready for module reload
```

### Data Flow Diagram

```
User Application
        │ (ioctl)
        ▼
VFS Layer
        │ (file operation dispatch)
        ▼
gpio_unlocked_ioctl()
        │
        ├─→ GPIO_LED_ON?
        │       │
        │       ▼
        │   gpio_set_value(GPIO 60, HIGH)
        │       │
        │       ▼
        │   GPIO Controller (Hardware)
        │
        └─→ GPIO_GET_BTN?
                │
                ▼
            gpio_get_value(GPIO 48)
                │
                ▼
            copy_to_user() [return to user app]

Hardware → Interrupt
        │
        ▼
CPU Routes to ISR
        │
        ▼
button_isr()
        │
        ├─→ atomic_set(&event_flag, 1)
        │
        └─→ wake_up_interruptible(&gpio_wq)
                │
                ▼
            Main Thread Wakes Up
                │
                ├─→ Create Slave Threads (if needed)
                │
                └─→ Wait again
```

### Synchronization Diagram

```
Timeline of Events:

T0: System Init
    ├─→ gpio_driver_init() called
    ├─→ Main thread created and sleeping (waiting for event)
    └─→ Slave threads NOT created yet

T1: Button Pressed
    ├─→ Hardware interrupt
    ├─→ button_isr() called
    │   ├─→ event_flag = 1 (atomic)
    │   └─→ wake_up_interruptible()
    └─→ ISR returns

T2: Main Thread Wakes
    ├─→ wait_event_interruptible() returns
    ├─→ Check: slave threads exist? NO
    ├─→ Create slave_thread_1
    ├─→ Create slave_thread_2
    ├─→ Both threads start running
    └─→ event_flag = 0 (reset)

T3: Slave Threads Running
    Both threads run concurrently:
    
    Slave_1                      Slave_2
    ├─→ Lock                     ├─→ Lock (waits)
    ├─→ Increment counter        │
    ├─→ Unlock                   ├─→ Lock (acquired)
    ├─→ Sleep 10ms               ├─→ Increment counter
                                 ├─→ Unlock
                                 └─→ Sleep 10ms

T4: Button Pressed Again
    ├─→ Hardware interrupt
    ├─→ button_isr() called
    │   ├─→ event_flag = 1
    │   └─→ wake_up_interruptible()
    └─→ Main thread wakes again

T5: Main Thread Wakes Again
    ├─→ Check: slave threads exist? YES
    ├─→ Don't create new ones
    ├─→ Reset event_flag = 0
    └─→ Wait again

T6: System Shutdown
    ├─→ gpio_driver_exit() called
    ├─→ kthread_stop(main)
    ├─→ kthread_stop(slave_1)
    ├─→ kthread_stop(slave_2)
    └─→ All resources freed
```

### Critical Sections & Synchronization

```
Shared Resource: thread_args (integer counter)

Protected by: spinlock (my_lock)

Access Pattern:
┌─────────────────────────────────────────┐
│ Slave Thread 1                          │
│                                         │
│ while (!kthread_should_stop()) {        │
│     spin_lock(&my_lock);       ◄─┐      │
│     thread_args++;             │        │
│     spin_unlock(&my_lock);     ┘        │
│     msleep(10);                         │
│ }                                       │
└─────────────────────────────────────────┘
        ▲                      ▲
        │ (Mutual             │ (Can be
        │  Exclusion)         interrupted
        │                     here)
┌─────────────────────────────────────────┐
│ Slave Thread 2                          │
│                                         │
│ while (!kthread_should_stop()) {        │
│     spin_lock(&my_lock);       ◄────┐   │
│     thread_args++;                  │   │
│     spin_unlock(&my_lock);     ────-┘   │
│     msleep(10);                         │
│ }                                       │ 
─────────────────────────────────────────-┘

Without spinlock: Race condition!
├─→ T1: Thread1 reads thread_args = 5
├─→ T2: Thread2 reads thread_args = 5
├─→ T3: Thread1 increments to 6, stores
├─→ T4: Thread2 increments to 6, stores
└─→ Result: thread_args = 6 (should be 7)

With spinlock: No race condition
├─→ T1: Thread1 locks, reads 5, increments, stores 6, unlocks
├─→ T2: Thread2 waits for lock
├─→ T3: Thread2 locks, reads 6, increments, stores 7, unlocks
└─→ Result: thread_args = 7 (correct)
```

---