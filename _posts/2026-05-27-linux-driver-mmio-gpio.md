---
title: MMIO-Access (GPIO- Interrupt Module)
date: 2026-05-27
categories: [Projects]
tags: [projects, gpio, interrupt]
---

Memory-mapped I/O (MMIO) is a method for accessing hardware registers by mapping physical device addresses into the processor's virtual address space. The GPIO controller exposes several registers at fixed physical addresses. This example directly manipulates the GPIO controller registers instead of using the higher-level GPIO framework, which makes the code hardware-specific but simple for learning low-level GPIO access.

# Overview

- `main.c` is a Linux kernel module source file.
- It performs memory-mapped I/O on a GPIO controller.
- The module turns a red LED on by configuring the GPIO pin as output and setting it high.
- The `Makefile` is a kernel module build script with support for cross-compiling for ARM and building on the host.

---
## Source Code: `main.c` 
[Github Code ](https://github.com/rajmahto9876/MMIO-INT-Driver)

- `linux/module.h`: kernel module infrastructure.
- `linux/io.h`: I/O memory mapping helpers, such as `ioremap`, `readl`, and `writel`.
  - The driver uses `ioremap` to create a virtual pointer to the GPIO register block.
   - `gpio_request` to requests ownership of a GPIO from the Linux GPIO subsystem so no other driver uses it simultaneously.
   - `gpio_to_irq` converts a Linux GPIO number into a Linux virtual IRQ number for interrupt handling.
   - `request_irq` registers ISR with the Linux IRQ subsystem for a specific IRQ line.
   - `kthread_create` creates a kernel thread but does NOT start executing it immediately.
   - `kthread_stop` signals a kernel thread to stop and waits until the thread exits safely.
   - `wake_up_process` changes a sleeping/stopped task state to runnable so the scheduler can execute it.
   - `readl` reads 32-bit register values, and `writel` writes 32-bit values.
- `linux/errno.h`: standard kernel error codes.
- `linux/gpio.h`: GPIO helper definitions and types.

### Macros (Address and Offset)

```c
#define GPIO1_BASE                      0x4804C000
#define GPIO_SIZE                       0x1000

/* GPIO Registers */
/* ARM Cortex-A8 Interrupts TRM */
#define GPIO1_IRQ                       98
#define GPIO_RISINGEDGE_OFFSET          0x148
#define GPIO_FALLINGDETECT_OFFSET       0x14C

#define GPIO_IRQSTATUS_0_OFFSET         0x2C
#define GPIO_IRQSTATUS_SET_0_OFFSET     0x34

#define GPIO_OE_REGISTER_OFFSET         0x134
#define GPIO_DATAIN_OFFSET              0x138
#define GPIO_DATAOUT                    0x13C

#define GPIO_INTRRUPT_PIN               12 //(GPIO1_12)
#define GPIO_TO_IRQ_PIN                 ((1*32) + GPIO_INTRRUPT_PIN)

#define RED_LED                         13

#define SET_BIT(value, pin)    ((value) | (1U << (pin)))
#define CLEAR_BIT(value, pin)  ((value) & ~(1U << (pin)))
#define TOGGLE_BIT(value, pin) (value^=  (1<<(pin)))
```

- `GPIO1_BASE` is the base physical address for GPIO1.
- `GPIO_BASE_SIZE` is the size of the mapped region.
- `GPIO_OE_REGISTER_OFFSET`, `GPIO_DATAIN_OFFSET`, and `GPIO_DATAOUT` are offsets within the GPIO register block.
- `RED_LED` is the GPIO pin number used for the LED.
- `SET_BIT` and `CLEAR_BIT` are helper macros for bit-based register updates.

### Global pointer

```c
static void __iomem *gpio_base = NULL;
static struct task_struct *kthread_handle = NULL;
atomic_t condition = ATOMIC_INIT(0);
```

- `gpio_base` stores the mapped virtual address for the GPIO registers.
- `__iomem` indicates that the pointer refers to I/O memory.

### Module init function

```c

static int __init my_driver_init(void)
{
    u32 val;
    u32 rValue = 0;
    int ret = 0;

    pr_info("Driver Init\n");
    gpio_base = ioremap(GPIO1_BASE, GPIO_SIZE);
    if (gpio_base == NULL)
    {
        pr_err("ioremap failed\n");
        return -ENOMEM;
    }

    /*
        Kthread to Do Some Task
    */
    ret = gpio_thread_init();
    if(ret< 0)
    {
        iounmap(gpio_base);
        pr_err("ioremap failed\n");
        return -ENOMEM;
    }
    /*
        Set Gpio1_13 as Output and clear it.
    */
    rValue = readl(gpio_base + GPIO_OE_REGISTER_OFFSET);
    pr_info("Register GPIO_OE_REGISTER_OFFSET value: 0x%02x\n", rValue);
    writel(CLEAR_BIT(rValue, RED_LED), gpio_base + GPIO_OE_REGISTER_OFFSET);

    rValue = readl(gpio_base + GPIO_DATAIN_OFFSET);
    pr_info("Register GPIO_DATAIN_OFFSET value: 0x%02x\n", rValue);
    writel(CLEAR_BIT(rValue, RED_LED), gpio_base + GPIO_DATAOUT);

    /*
        Set Gpio1_12 as Input Interrupt Mode
        Rising Edge Detection.
    */
    rValue = readl(gpio_base + GPIO_OE_REGISTER_OFFSET);
    pr_info("Register GPIO_OE_REGISTER_OFFSET value: 0x%02x\n", rValue);
    writel(SET_BIT(rValue, GPIO_INTRRUPT_PIN), gpio_base + GPIO_OE_REGISTER_OFFSET);

    val = readl(gpio_base + GPIO_RISINGEDGE_OFFSET);
    pr_info("Register GPIO_RISINGEDGE_OFFSET value: 0x%02x\n", val);

    writel(SET_BIT(val, GPIO_INTRRUPT_PIN), gpio_base + GPIO_RISINGEDGE_OFFSET);

    /*
     * STEP 4:
     * Enable interrupt generation
     */

    val = readl(gpio_base + GPIO_IRQSTATUS_SET_0_OFFSET);
    pr_info("Register GPIO_IRQSTATUS_SET_0_OFFSET value: 0x%02x\n", val);
    writel(SET_BIT(val, GPIO_INTRRUPT_PIN), gpio_base + GPIO_IRQSTATUS_SET_0_OFFSET);


    ret = gpio_request(GPIO_TO_IRQ_PIN, "gpio_intr");
    if (ret)
    {
        pr_err("gpio_request failed\n");
        goto end;
    }
    
    pr_info("Interrupt enabled in GPIO module\n");
    ret = request_irq(gpio_to_irq(GPIO_TO_IRQ_PIN), gpio_irq_handler, IRQF_TRIGGER_RISING,
                      "my_gpio_irq", &gpio_base);

    if (ret!= 0) 
    {
        pr_err("request_irq failed\n");
        goto end;
    }

    pr_info("ISR Registered\n");
    return 0;

end:
    if(kthread_handle!= NULL)
    {
        kthread_stop(kthread_handle);
    }
    iounmap(gpio_base);
    return ret;
}

```

- `gpio_test_init` is called when the module is inserted.
- `ioremap` maps the physical GPIO register bank into the kernel address space.
- If mapping fails, the code logs an error and returns `-ENOMEM`.
- It reads the output enable register and clears the bit for pin `RED_LED`, making the pin an output.
- Then it reads the output value register, sets the LED bit, and writes it back to turn the LED on.

### Module exit function

```c
static void __exit my_driver_exit(void)
{
    u32 rValue = 0;
    pr_info("Driver Exit\n");
    if(kthread_handle!= NULL)
    {
        kthread_stop(kthread_handle);
    }

    /* setting pin to Gnd. */
    rValue = readl(gpio_base + GPIO_DATAOUT);
    rValue = rValue & ~(1<<RED_LED);
    rValue = rValue & ~(1<<GPIO_INTRRUPT_PIN);
    writel(rValue, gpio_base + GPIO_DATAOUT);

    /* setting pin to INPUT Mode. */
    rValue = readl(gpio_base + GPIO_OE_REGISTER_OFFSET);
    writel(SET_BIT(rValue, RED_LED), gpio_base + GPIO_OE_REGISTER_OFFSET);
    writel(SET_BIT(rValue, GPIO_INTRRUPT_PIN), gpio_base + GPIO_OE_REGISTER_OFFSET);

    free_irq(gpio_to_irq(GPIO_TO_IRQ_PIN), &gpio_base);
    iounmap(gpio_base);

    pr_info("Resources Freed\n");
}
```

- `gpio_test_exit` is called when the module is removed.
- It clears the output bit for the red LED to turn it off.
- It sets the pin back to input mode by setting the bit in the output-enable register.
- `iounmap` releases the mapped I/O region.
- It logs an exit message.

### Module macros

```c
module_init(gpio_test_init);
module_exit(gpio_test_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("RAJ KUMAR");
MODULE_DESCRIPTION("IO-REMAP MODULE");
```

- `module_init` / `module_exit` register the init and cleanup functions.
- `MODULE_LICENSE("GPL")` declares the module license and avoids tainting the kernel.
- `MODULE_AUTHOR` and `MODULE_DESCRIPTION` provide metadata.

---

## Interrupt control logic

The actual `main.c` implementation also configures GPIO interrupts and uses a worker thread to handle LED toggling safely outside interrupt context.

- `GPIO_INTRRUPT_PIN` is `12`, which means the driver uses GPIO1_12 as the interrupt input.
- `GPIO_TO_IRQ_PIN` computes the Linux IRQ number for that GPIO pin with `gpio_to_irq`.
- The code enables rising-edge detection on the pin by setting the bit in `GPIO_RISINGEDGE_OFFSET`.
- It enables the GPIO interrupt generator by writing the pin bit to `GPIO_IRQSTATUS_SET_0_OFFSET`.
- `gpio_request(GPIO_TO_IRQ_PIN, "gpio_intr")` reserves the GPIO IRQ line.
- `request_irq(..., gpio_irq_handler, IRQF_TRIGGER_RISING, "my_gpio_irq", &gpio_base)` registers the ISR.

### Interrupt Service Routine (ISR)

```c
static irqreturn_t gpio_irq_handler(int irq, void *dev_id)
{
    u32 status;
    u32 flag;

    pr_info("ISR Triggered\n");
    status = readl(gpio_base + GPIO_IRQSTATUS_0_OFFSET);
    pr_info("IRQSTATUS = 0x%x\n", status);

    atomic_set(&condition, 1);

    flag = readl(gpio_base + GPIO_IRQSTATUS_0_OFFSET);
    writel(SET_BIT(flag, GPIO_INTRRUPT_PIN), gpio_base + GPIO_IRQSTATUS_0_OFFSET);
    pr_info("Interrupt Cleared\n");

    return IRQ_HANDLED;
}
```

- The ISR reads the interrupt status register to determine the source.
- It signals the worker thread by setting `condition` to `1`.
- It clears the GPIO interrupt by writing the pin bit back to the status register.
- It returns `IRQ_HANDLED` to tell the kernel the interrupt was serviced.

### kthread Unblocking:

- The driver creates a kernel thread with `kthread_create` and wakes it with `wake_up_process`.
- `kthread_thread_fn` loops until `kthread_should_stop()` returns true.
- When `condition` becomes `1`, the thread toggles the red LED pin (`RED_LED` = 13) by reading and writing `GPIO_DATAOUT`.

### Cleanup and exit

- `free_irq(gpio_to_irq(GPIO_TO_IRQ_PIN), &gpio_base)` releases the IRQ reservation.
- The driver resets the LED and input pin state before unmapping the GPIO registers.
- `iounmap(gpio_base)` releases the MMIO mapping at module unload.

---

## Makefile

The `Makefile` is designed to build the kernel module using the Linux kernel build system.

### Module build variables

```makefile
obj-m := gpio.o
gpio-objs += main.o
ARCH=arm
CROSS_COMPILE=arm-linux-gnueabihf-
```

- `obj-m := gpio.o` tells the kernel build system that this directory builds a loadable module named `gpio.ko`.
- `gpio-objs += main.o` indicates that `main.c` is compiled into the `gpio` module.
- `ARCH=arm` sets the target architecture to ARM.
- `CROSS_COMPILE=arm-linux-gnueabihf-` sets the cross-compiler prefix for building on a host machine targeting ARM.

### Kernel source directories

```makefile
KERN_DIR =<path>/linux
HOST_KERN_DIR = /lib/modules/$(shell uname -r)/build/
```

- `KERN_DIR` should point to the ARM kernel source tree for cross-compilation.
- `HOST_KERN_DIR` points to the host machine's installed kernel build directory.

### Compiler flags

```makefile
ccflags-y := -DMODULE_DEBUG
```

- Adds `-DMODULE_DEBUG` to kernel module compilation flags.
- This can be used in source code to enable debug-specific code paths.

### Build targets

```makefile
all:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) modules

clean:
	make ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(KERN_DIR) M=$(PWD) clean
```

- `all` builds the module by invoking the kernel build system in `$(KERN_DIR)`.
- `M=$(PWD)` tells the kernel build system to build the module from the current directory.
- `clean` removes generated build artifacts.

## Expected Outputs
### Insertion of Module

```text
sudo insmod gpio_intr.ko 
sudo dmesg

Expected:

[ 1596.379788] Driver Init
[ 1596.380061] Statring Threads
[ 1596.380078] Register GPIO_OE_REGISTER_OFFSET value: 0xf61fdfff
[ 1596.380084] Register GPIO_DATAIN_OFFSET value: 0x36010300
[ 1596.380090] Register GPIO_OE_REGISTER_OFFSET value: 0xf61fdfff
[ 1596.380095] Register GPIO_RISINGEDGE_OFFSET value: 0x00
[ 1596.380101] Register GPIO_IRQSTATUS_SET_0_OFFSET value: 0x00
[ 1596.380125] Interrupt enabled in GPIO module
[ 1596.380190] ISR Registered
[ 1596.388438] In Thread 
```

### IRQ Trigger
```text 
sudo dmesg

[ 1729.723318] ISR Triggered
[ 1729.723322] IRQSTATUS = 0x1000
[ 1729.723326] Interrupt Cleared
[ 1729.723347] ISR Triggered
[ 1729.723352] IRQSTATUS = 0x0
[ 1729.723356] Interrupt Cleared
```

### Remove Module
```text
sudo rmmod gpio_intr 
sudo dmesg

Expected:

[ 1810.222209] Driver Exit
[ 1810.227830] Resources Freed
```