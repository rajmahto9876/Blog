---
title: Understanding ARM (Cortex-M) Startup Code
date: 2026-06-15
categories: [arm]
tags: [arm, start-up, cortex-m]
---

Startup code is the first code executed when a Cortex-M microcontroller powers on or resets. It runs before the main application and is responsible for initializing the processor, memory, and runtime environment.

The startup code typically includes:
- **Vector table setup** - Interrupt handlers and reset handler locations
- **Stack initialization** - Setting up the main stack pointer (MSP)
- **Memory initialization** - Copying initialized data and zeroing BSS section
- **Clock configuration** - Setting up system clocks
- **Runtime initialization** - Preparing C runtime environment
- **Jump to main** - Transferring control to the main application

## The Vector Table

The vector table is a critical data structure that contains the addresses of interrupt handlers and the initial stack pointer. On Cortex-M processors, the vector table is located at the start of the addressable memory space, typically at address 0x00000000.

### Vector Table Structure

```c
// Typical vector table entry
typedef struct
{
    uint32_t *initial_sp;           // Vector[0]: Initial Stack Pointer
    uint32_t *reset_handler;        // Vector[1]: Reset Handler
    uint32_t *nmi_handler;          // Vector[2]: NMI Handler
    uint32_t *hard_fault_handler;   // Vector[3]: Hard Fault Handler
    uint32_t *mem_manage_fault;     // Vector[4]: Memory Manage Fault
    uint32_t *bus_fault_handler;    // Vector[5]: Bus Fault Handler
    uint32_t *usage_fault_handler;  // Vector[6]: Usage Fault Handler
    // ... more exception vectors
    uint32_t *irq_handlers[240];    // External IRQ handlers (varies by device)
} vector_table_t;
```

### Vector Table Attributes

Each entry in the vector table is a 32-bit value. The LSB (bit 0) must always be 1, indicating Thumb mode code. For example:
- Handler address: `0x08001234` → Vector entry: `0x08001235` (with Thumb bit set)

The first entry is special—it contains the initial stack pointer address, not a handler address.

## Startup Code Flow

Here's a typical startup sequence:

```
┌─────────────────────────────────────┐
│   Power On / Reset                   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   CPU reads Vector[0] → MSP         │
│   CPU reads Vector[1] → PC (Reset)  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Reset Handler Executes            │
│   - Copy initialized data (RW)      │
│   - Zero BSS section (RW)           │
│   - Call system init functions      │
│   - Call C constructors             │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Jump to main()                     │
└─────────────────────────────────────┘
```

## Memory Sections

Cortex-M applications typically use several memory sections:

### 1. **RO (Read-Only) Section** - Flash Memory
Contains code and constant data that doesn't change at runtime.

```c
// Linker script notation
FLASH (rx) : ORIGIN = 0x00000000, LENGTH = 256K
```

### 2. **RW (Read-Write) Section** - RAM
Contains initialized variables copied from flash at startup.

```c
// Example initialized variable
int counter = 42;  // This value is stored in flash and copied to RAM
```

### 3. **ZI (Zero-Initialized) Section** - RAM
Contains uninitialized variables that must be zeroed.

```c
// Example uninitialized variable
int buffer[1024];  // This must be zeroed at startup
```

## A Comprehensive Startup Code Example

Here's a complete startup code implementation for a Cortex-M4 processor:

```c
// startup.c - Cortex-M4 Startup Code

#include <stdint.h>

// Forward declarations
void Reset_Handler(void);
void Default_Handler(void);
void NMI_Handler(void) __attribute__((weak, alias("Default_Handler")));
void HardFault_Handler(void) __attribute__((weak, alias("Default_Handler")));
void MemManage_Handler(void) __attribute__((weak, alias("Default_Handler")));
void BusFault_Handler(void) __attribute__((weak, alias("Default_Handler")));
void UsageFault_Handler(void) __attribute__((weak, alias("Default_Handler")));
void SVCall_Handler(void) __attribute__((weak, alias("Default_Handler")));
void DebugMon_Handler(void) __attribute__((weak, alias("Default_Handler")));
void PendSV_Handler(void) __attribute__((weak, alias("Default_Handler")));
void SysTick_Handler(void) __attribute__((weak, alias("Default_Handler")));

// External symbols from linker script
extern uint32_t __etext;      // End of initialized data in flash
extern uint32_t __data_start__; // Start of data section in RAM
extern uint32_t __data_end__;   // End of data section in RAM
extern uint32_t __bss_start__;  // Start of BSS section
extern uint32_t __bss_end__;    // End of BSS section
extern uint32_t __StackTop;     // Top of stack

// Main function
extern int main(void);

// Vector Table
__attribute__((section(".vectors"))) 
const uint32_t g_pfnVectors[] = {
    // Stack pointer value
    (uint32_t)&__StackTop,
    // Reset handler
    (uint32_t)&Reset_Handler,
    // Core exceptions
    (uint32_t)&NMI_Handler,
    (uint32_t)&HardFault_Handler,
    (uint32_t)&MemManage_Handler,
    (uint32_t)&BusFault_Handler,
    (uint32_t)&UsageFault_Handler,
    0,  // Reserved
    0,  // Reserved
    0,  // Reserved
    0,  // Reserved
    (uint32_t)&SVCall_Handler,
    (uint32_t)&DebugMon_Handler,
    0,  // Reserved
    (uint32_t)&PendSV_Handler,
    (uint32_t)&SysTick_Handler,
    // External interrupts (example for M4)
    (uint32_t)&Default_Handler,  // IRQ0
    (uint32_t)&Default_Handler,  // IRQ1
    // ... more IRQ handlers as needed
};

/**
 * Reset Handler
 */
void Reset_Handler(void)
{
    // Copy initialized data from flash to RAM
    uint32_t *source = &__etext;
    uint32_t *dest = &__data_start__;
    
    while (dest < &__data_end__)
    {
        *dest++ = *source++;
    }
    
    // Zero the BSS section
    dest = &__bss_start__;
    while (dest < &__bss_end__)
    {
        *dest++ = 0;
    }
        
    // Initialize system clocks and peripherals
    SystemInit();
    
    // Call main
    main();
    
    // Infinite loop if main returns
    while (1)
    {
        __asm("NOP");
    }
}

/**
 * Default Handler
 */
void Default_Handler(void)
{
    while (1) 
    {
        __asm("NOP");
    }
}

/**
 * System Initialization
 * Called before main() to initialize clocks, peripherals, etc.
 */
void SystemInit(void)
{
    // Enable FPU (Floating Point Unit) if present
    SCB->CPACR |= ((3UL << 10*2) | (3UL << 11*2));
    
    // Configure system clocks

    // Initialize any global peripherals needed before main
}
```

## Linker Script Considerations

The linker script defines the memory layout and provides symbols used by startup code:

```ld
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x00000000, LENGTH = 256K
    RAM (rwx)   : ORIGIN = 0x20000000, LENGTH = 64K
}

SECTIONS
{
    .vectors :
    {
        KEEP(*(.vectors))
    } > FLASH

    .text :
    {
        *(.text*)
        *(.rodata*)
    } > FLASH

    .data :
    {
        __data_start__ = .;
        *(.data*)
        __data_end__ = .;
    } > RAM AT > FLASH
    
    __etext = LOADADDR(.data);

    .bss :
    {
        __bss_start__ = .;
        *(.bss*)
        *(COMMON)
        __bss_end__ = .;
    } > RAM

    .stack :
    {
        __StackLimit = .;
        . += 0x1000;  // 4KB stack
        __StackTop = .;
    } > RAM
}
```

## Key Concepts

### 1. **Thumb Bit Setting**
The LSB of all handler addresses must be 1 to indicate Thumb code:

```c
void (*handler_ptr)(void) = (void (*)(void))(address | 0x1);
```

### 2. **Memory Initialization Order**
The order matters for correct program behavior:

```
1. Initialize Data (RW section) - Copy from flash to RAM
2. Zero BSS (ZI section) - Clear RAM for uninitialized variables
3. Call SystemInit() - Set up clocks and peripherals
4. Call main() - Enter application code
```

### 3. **Stack Growth Direction**
Cortex-M stacks grow downward (from higher to lower addresses). The stack pointer starts at the highest address of the stack region.

### 4. **Exception Nesting**
Cortex-M supports nested interrupts. The hardware automatically saves registers on the stack when an exception occurs.

## References

- ARM Cortex-M Architecture Reference Manual
- Microcontroller device datasheets
- GNU Linker script documentation
- ARM EABI (Embedded Application Binary Interface)