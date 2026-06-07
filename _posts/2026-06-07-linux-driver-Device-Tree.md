---
title: Device Tree
date: 2026-06-07
categories: [Linux-Driver-Development]
tags: [linux, Device Tree , QEMU]
---

# Introduction

The Linux Device Tree (DT) is a powerful abstraction layer that separates hardware configuration from kernel code. Instead of hardcoding hardware details into the Linux kernel, the device tree provides a flexible, standardized way to describe hardware resources, peripherals, and their interconnections.

## Device Tree
A Device Tree is a text-based data structure that describes the hardware components of a computer system in a platform-independent, hierarchical format. It defines:

- **CPU and memory configurations**
- **Peripheral devices** and their memory-mapped I/O addresses
- **Interrupt routing** and controller information
- **Device-specific properties** (clocks, speeds, names, buffers, etc.)
- **Power management** and clock specifications
- **Pin multiplexing** and GPIO configurations

The device tree is compiled into a binary format (Device Tree Blob - DTB) and passed to the kernel at boot time, eliminating the need to recompile the kernel for different hardware variations.

## Use Cases

### 1. **Hardware Abstraction**
Device trees decouple hardware configuration from kernel code. Used to modify hardware without touching kernel source:

```
Traditional Approach:
Hardware Change → Modify Kernel Code → Recompile → Deploy

Device Tree Approach:
Hardware Change → Update Device Tree → Compile DTB → Deploy
```

### 2. **Portability**
A single Linux kernel binary can run on multiple hardware platforms with different device trees.

### 3. **Flexibility**
Device overlays allow adding or modifying devices dynamically without full system reboot, useful for hot-pluggable devices and development scenarios.

### 4. **Standardization**
Device trees follow an open specification (Devicetree Specification), making hardware descriptions consistent across different Linux distributions and architectures.

## Device Tree Structure and Syntax

Device trees use a hierarchical, text-based format (DTS - Device Tree Source).
```
/dts-v1/;                          // DTS version declaration

/ {                                 // Root node
    compatible = "your,device_name";   // System compatibility
    #address-cells = <0x02>;        // Address cell width
    #size-cells = <0x02>;           // Size cell width
    interrupt-parent = <0x8001>;    // Global interrupt controller
    
    node_name@address {             // Device node
        compatible = "your,driver_name";
        reg = <address size>;       // Memory address and size
        interrupts = <irq_info>;    // Interrupt configuration
        property = "value";         // Custom properties
    };
};
```

### Key Components:

**1. Compatible Property**
```dts
compatible = "raj,test-device";
```
This string tells the kernel which driver handles this device. The kernel uses this for driver matching.

**2. Register (reg) Property**
```dts
reg = <0x00 0xa000a00 0x00 0x200>;
```
Specifies memory-mapped I/O addresses and sizes. With `#address-cells = <2>` and `#size-cells = <2>`, this means:
- Address: 0x00a000a00
- Size: 0x200 bytes

**3. Interrupt Property**
```dts
interrupts = <0x00 0x15 0x01>;
```
Defines interrupt information for the device. The format depends on the interrupt controller.

**4. Custom Properties**
Properties can store device-specific data accessible to drivers:
```dts
device-name = "test-device";
buffer-size = <256>;
```

## Real-World Example: QEMU Platform Driver

Let's examine the provided QEMU device tree and platform driver to understand practical usage:

### Gtihub Code:
The main kernel driver is implemented in `main.c`<br>
[ARMv8/02_Platform_driver](https://github.com/rajmahto9876/linux-device-driver-armv8.git)


### Device Tree Configuration (qemu.dts)

```dts
/ {
    compatible = "linux,dummy-virt";
    #address-cells = <0x02>;
    #size-cells = <0x02>;
    interrupt-parent = <0x8001>;
    
    cpus {
        #address-cells = <0x01>;
        #size-cells = <0x00>;
        cpu@0 {
            device_type = "cpu";
            reg = <0x00>;
            compatible = "arm,cortex-a57";
        };
    };
    
    virtio_mmio@a000a00 {
        reg = <0x00 0xa000a00 0x00 0x200>;
        compatible = "virtio,mmio";
        interrupts = <0x00 0x15 0x01>;
        dma-coherent;
    };
    
    // More virtio devices...
    
    test_device {
    	compatible = "raj,test-device";
    	status = "okay";
    	reg = <0x00 0x10000000 0x00 0x1000>, <0x00 0x20000000 0x00 0x1000>;
    	reg-names = "control", "data";

    	interrupts = <0x00 0x1c 0x01>, <0x00 0x1d 0x01>;
    	interrupt-names = "rx_irq", "tx_irq";

    	buffer-size = <512>;
    	device-name = "raj_qemu";
    };
};
```

**What This Defines:**
- ARM Cortex-A57 CPU for ARMv8 architecture
- Multiple VIRTIO MMIO devices for I/O virtualization
- A Custom Configuration of custom device
- Interrupt routing through controller 0x8001

### Platform Driver Implementation (main.c)

The platform driver leverages the device tree to extract hardware information:

```c
static int test_probe(struct platform_device *pDev)
{
    int ret = 0;
    struct resource *res_control = NULL;
    struct resource *res_data    = NULL;
    struct test_dtb_data *dtc_data = NULL;

    pr_info("TEST: Probe called\n");
    dtc_data = (struct test_dtb_data*)devm_kzalloc(&pDev->dev, sizeof(struct test_dtb_data), GFP_KERNEL);
    if(dtc_data == NULL)
    {
        pr_err("devm_kzalloc Failed \n");
        return -ENOMEM;
    }
    
    res_control = platform_get_resource_byname(pDev, IORESOURCE_MEM, reg_control_name);
    if(!(res_control))
    {
        pr_err("platform_get_resource_byname control \n");
        return -EINVAL;
    }
    pr_info("start=%pa end=%pa\n", &res_control->start, &res_control->end);

    dtc_data->base_control = devm_ioremap_resource(&pDev->dev, res_control);
    if(!(dtc_data->base_control))
    {
        pr_err("devm_ioremap base_control \n");
        return -ENODEV;
    }

    res_data = platform_get_resource_byname(pDev, IORESOURCE_MEM, reg_data_name);
    if(!(res_data))
    {
        pr_err("platform_get_resource_byname data \n");
        return -EINVAL;
    }
    pr_info("start=%pa end=%pa\n", &res_data->start, &res_data->end);

    dtc_data->base_data = devm_ioremap_resource(&pDev->dev, res_data);
    if(!(dtc_data->base_data))
    {
        pr_err("devm_ioremap base_data \n");
        return -ENODEV;
    }

    dtc_data->irq_rx = platform_get_irq_byname(pDev, rx_interrupt_name);
    if(ret!= 0)
    {
        pr_err("platform_get_irq_byname rx_irq \n");
        return ret;
    }
    pr_info("irq_name_rx =>[%d]\n", dtc_data->irq_rx);

    dtc_data->irq_tx = platform_get_irq_byname(pDev, tx_interrupt_name);
    if(ret!= 0)
    {
        pr_err("platform_get_irq_byname tx_irq \n");
        return ret;
    }
    pr_info("irq_name_tx =>[%d]\n", dtc_data->irq_tx);

    ret = of_property_read_string(pDev->dev.of_node, "device-name", &dtc_data->dvc_name);
    if(ret!= 0)
    {
        pr_err("of_property_read_string string \n");
        return ret;
    }

    pr_info("device - name=>[%s]\n", dtc_data->dvc_name);

    ret = of_property_read_u32(pDev->dev.of_node, "buffer-size", &dtc_data->buffer_size);
    if(ret!= 0)
    {
        pr_err("of_property_read_string int \n");
        return ret;
    }

    pr_info("device - buffer-size=>[%d]\n", dtc_data->buffer_size);

    pr_info("TEST: Probe Successfull\n");
    return 0;
}
```

**How It Works:**

1. **Driver Matching:** The kernel scans the device tree for nodes with `compatible = "raj,test-device"`
2. **Probe Callback:** When a match is found, `test_probe()` is called
3. **Resource Extraction:** The driver retrieves memory addresses, sizes, and interrupt numbers
4. **Property Reading:** Custom device tree properties are read using `of_property_read_*()` functions
5. **Memory Mapping:** I/O memory is mapped to kernel virtual addresses using `devm_ioremap_resource()`

## Device Tree Compilation

Device trees must be compiled from human-readable DTS format to binary DTB format:

```bash
dtc -I dts -O dtb -o qemu.dtb qemu.dts
```

The Makefile includes this target:

```text
dtbo:
    dtc -@ -I dts -O dtb -o qemu.dtbo qemu.dts
```

**Compilation Flags:**
- `-@`: Includes phandle references for device tree overlays
- `-I dts`: Input format is Device Tree Source
- `-O dtb`: Output format is Device Tree Binary

## Device Tree Bindings and Drivers

Device Tree Bindings document how hardware should be described for specific drivers. They define:

- **Required properties** that must be present
- **Optional properties** for configuration
- **Constraints** on property values
- **Examples** of proper usage

### Binding for Our Test Device:

```
Required Properties:
- compatible: Must be "raj,test-device"
- reg: Two memory regions named "control" and "data"
- interrupts-names: "rx_irq", "tx_irq"

Optional Properties:
- device-name: Human-readable device name (string)
- buffer-size: Buffer size in bytes (integer)
```

## Platform Driver Integration

The Linux kernel provides the platform driver for DT-based drivers:

```c
static const struct of_device_id test_of_match[] = {
    { .compatible = "raj,test-device" },
    { }
};
MODULE_DEVICE_TABLE(of, test_of_match);

static struct platform_driver test_driver = {
    .probe  = test_probe,
    .remove = test_remove,
    .driver = {
        .name = "test_driver",
        .of_match_table = test_of_match,
    },
};

module_platform_driver(test_driver);
```

**Framework Benefits:**
- Automatic driver discovery and loading
- Device lifecycle management
- Resource allocation and deallocation
- Error handling standardization

## Common Device Tree Properties

| Property | Type | Purpose |
|----------|------|---------|
| `compatible` | String | Driver matching identifier |
| `reg` | Integer array | Memory address and size |
| `interrupts` | Integer array | Interrupt configuration |
| `status` | String | Device status (okay/disabled) |
| `#address-cells` | Integer | Address format specifier |
| `#size-cells` | Integer | Size format specifier |
| `ranges` | Integer array | Address translation |
| `clocks` | Phandle | Clock references |
| `power-domains` | Phandle | Power management domain |
| `dma-coherent` | Flag | DMA cache coherency hint |

## Advanced Device Tree Features

### 1. **Device Tree Overlays (DTO)**
Dynamically add devices without recompiling the entire tree:

```dts
/dts-v1/;
/plugin/;

&root {
    new_device@address {
        compatible = "vendor,device";
        reg = <...>;
    };
};
```

### 2. **Phandles and References**
Reference other nodes:

```dts
&gic {
    interrupts = <...>;
};
```

### 3. **Labels and Aliases**
Simplify addressing:

```dts
gic: interrupt-controller@8001 {
    ...
};

aliases {
    serial0 = "/uart@9000";
};
```

## Debugging Device Trees

### Viewing System Device Tree
```bash
# At runtime
cat /proc/device-tree/

# Decompile DTB to DTS
dtc -I dtb -O dts -o output.dts system.dtb

# Check device tree properties
od -A x -t x1z -v /proc/device-tree/device-name
```

### Kernel Logs
Device tree probe messages appear in kernel logs:
```
[   20.246091] platform: loading out-of-tree module taints kernel.
[   20.257615] TEST: Probe called
[   20.258470] start=0x0000000010000000 end=0x0000000010000fff
[   20.259463] start=0x0000000020000000 end=0x0000000020000fff
[   20.261753] irq_name_rx =>[26]
[   20.262645] irq_name_tx =>[27]
[   20.263190] device - name=>[raj_qemu]
[   20.263695] device - buffer-size=>[512]
[   20.264370] TEST: Probe Successfull

```
## References

- [Devicetree Specification](https://github.com/devicetree-org/devicetree-specification)
- [Linux Device Tree Bindings](https://www.kernel.org/doc/html/latest/devicetree/index.html)
- [Device Tree for Dummies](https://elinux.org/Device_Tree_for_Dummies)
- [Linux Platform Drivers](https://www.kernel.org/doc/html/latest/driver-api/driver-model/platform.html)

---

*This blog post was created based on a real QEMU platform driver implementation for ARMv8 architecture, demonstrating practical device tree usage in modern Linux systems.*
