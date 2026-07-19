---
title: "Cortex-M Memory Model"
date: 2026-07-19 01:00:00
categories: arm
tags: [arm, cortex-m, memory model, embedded]
---

When we write firmware for an ARM Cortex-M microcontroller, one of the most important ideas to understand is the memory model. It is the map that tells you where code lives, where variables live, how the CPU reaches peripherals, and how the system protects memory regions. In embedded systems, the memory model is not just a software concept. It is part of the hardware architecture itself.

This helps us in understanding:
- where your program code is executed from,
- where global variables and the stack are stored,
- how peripheral registers are accessed,
- why `volatile` matters,
- how the MPU helps enforce safety,
- and why some bugs only appear on one MCU family but not another.

---

## Memory Model

The memory model describes how the processor views the address space and how different memory regions behave. On Cortex-M, the CPU sees a single 32-bit address space, usually with a total range of 2**32 = 4 GB.

That address space is used for:

- instruction memory (flash/ROM),
- data memory (SRAM),
- peripheral registers,
- special system control blocks,
- debug and trace components,
- and sometimes vendor-specific memory like TCM or boot ROM.

The key point is that Cortex-M does not treat every address the same. Different regions have different properties:

- some are executable,
- some are writable,
- some are for device registers,
- some are strongly ordered,
- and some are protected by the MPU.

---

## The Cortex-M Address Space

A Cortex-M processor normally uses a 32-bit address space, which gives a total of:

`2^32 = 4,294,967,296 bytes ≈ 4 GB`

The exact mapping of those addresses depends on the MCU vendor and the specific Cortex-M core, but the standard ARM-defined regions are commonly used as follows:

```text
0x00000000 - 0x1FFFFFFF  Code region
0x20000000 - 0x3FFFFFFF  SRAM region
0x40000000 - 0x5FFFFFFF  Peripheral region
0x60000000 - 0x7FFFFFFF  External RAM region
0x80000000 - 0x9FFFFFFF  External device region
0xA0000000 - 0xBFFFFFFF  External device region
0xC0000000 - 0xDFFFFFFF  Private peripheral bus / special regions
0xE0000000 - 0xFFFFFFFF  System and vendor-specific regions
```

The important part is not the exact numbers in every case, but the idea that Cortex-M separates memory into classes with different meanings.

### Code Region

The region starting near `0x00000000` is commonly used for executable code, such as:

- flash memory,
- boot ROM,
- internal program memory,
- and sometimes external memory if the MCU supports it.

This is where the CPU fetches instructions from when the program runs. On many microcontrollers, the reset vector is stored at the beginning of the memory map, and the boot code jumps to the actual application entry point from there.

### SRAM Region

The SRAM region, typically starting at `0x20000000`, is used for:

- global and static variables,
- the stack,
- the heap,
- local variables in the call stack,
- and other runtime data.

This is usually the most critical region in embedded systems because it is where your program state lives. A stack overflow or corrupt memory write in SRAM can silently crash the system or cause mysterious bugs.

### Peripheral Region

The peripheral region starts around `0x40000000`. This is where memory-mapped peripheral registers live. Instead of having special instructions for each device, the CPU accesses them as ordinary memory locations.

Examples include:

- GPIO control registers,
- USART/UART registers,
- timers,
- ADC registers,
- DMA control registers,
- and interrupt controller registers.

This is why embedded C code often uses `volatile uint32_t*` pointers to access hardware registers.

---

## Memory Types 

Cortex-M does not only care about addresses. It also cares about memory attributes. In general, memory can be classified as:

- executable memory,
- data memory,
- device memory,
- normal memory,
- privileged-only memory,
- and execute-never memory.

This is especially important when the MPU is present.

### Normal Memory

Normal memory is the kind of memory you usually think of for RAM and flash. It is used for code and data. The processor may perform optimizations such as buffering or speculative reads, depending on the implementation.

For firmware developers, normal memory usually means:

- data can be read or written normally,
- instruction execution is allowed if the region is marked executable,
- and the CPU may use the memory system in a more conventional way.

### Device Memory

Device memory is used for peripherals and other hardware that must be accessed in a very controlled way. Access to device memory is often:

- non-cacheable,
- strongly ordered,
- and not subject to the same buffering optimizations as normal RAM.

This matters because a peripheral register must be read or written exactly when the software asks for it. If the core optimized accesses too aggressively, the hardware might not behave correctly.

---

## The Role of `volatile`

In embedded systems, a peripheral register is not just a place to store a value. It is a communication channel with hardware.

Consider code like this:

```c
volatile uint32_t *uart_status = (volatile uint32_t *)0x40011000;

while ((*uart_status & 0x20) == 0) 
{
    ;
}
```

The `volatile` keyword tells the compiler:

- do not optimize away repeated reads,
- do not assume the value is unchanged,
- and do not cache the value in a register.

Without `volatile`, the compiler might optimize the code in a way that breaks the hardware interaction. This is one of the first places where the Cortex-M memory model becomes visible in C code. `volatile` solves the problem of compiler optimization, but it does not solve everything. It does not create memory barriers automatically, and it does not guarantee ordering between unrelated hardware accesses on all architectures. In some cases, you need a memory barrier or a carefully designed access pattern.

For example, when working with DMA or shared memory, the software must ensure that writes are visible to the hardware and that the hardware sees them in the correct order.

---

## The Stack, Heap, and Program Layout

The memory model also includes the layout of your software image in memory. A typical Cortex-M application has sections such as:

```text
.text   -> executable code
.data   -> initialized global/static variables
.bss    -> zero-initialized global/static variables
.heap   -> dynamic memory allocation
.stack  -> call stack and interrupt stack
```

### `.text`

The `.text` section holds the machine instructions. It is usually placed in flash or ROM and is marked as executable.

### `.data`

The `.data` section contains initialized global variables. On startup, the loader copies this data from flash into SRAM so the program can access it at runtime.

### `.bss`

The `.bss` section contains globally or statically allocated variables that should start at zero. The startup code zeroes this region before `main()` begins.

### `.heap`

The heap is the region used by `malloc` and `free`. On bare-metal systems, it is often implemented by a simple allocator in SRAM.

### `.stack`

The stack is where function call frames, local variables, and interrupt context are stored. On Cortex-M, the stack grows downward by default, and the startup code initializes the initial stack pointer from the vector table.

A stack overflow is one of the most common failure modes in embedded software. If the heap and stack overlap or the stack grows beyond the allocated SRAM region, the program can corrupt memory in subtle ways.

---

## The Cortex-M Stack Model

The Cortex-M architecture uses a stack for exception handling and for ordinary function calls. When an exception occurs, the core pushes a hardware stack frame that contains important register values.

A typical exception frame includes:

```text
R0
R1
R2
R3
R12
LR
PC
xPSR
```

The exact frame layout can vary depending on the exception and the FPU state, but the idea is the same: the processor saves enough context so it can return later without losing execution state.

### MSP and PSP

Cortex-M has two stack pointers in many implementations:

- `MSP` (Main Stack Pointer)
- `PSP` (Process Stack Pointer)

The processor usually uses `MSP` in handler mode and `PSP` in thread mode. This separation is used by systems that want stronger control over privilege and stack usage, such as RTOS-based systems.

The memory model interacts with this because the stack regions themselves are just memory regions in SRAM. The processor simply uses them differently based on the current mode.

---

## Endianness

Cortex-M processors are generally little-endian. That means the least significant byte of a multi-byte value is stored at the lowest address.

Example:

```c
uint32_t value = 0x11223344;
```

In memory, it is stored as:

```text
address+0: 0x44
address+1: 0x33
address+2: 0x22
address+3: 0x11
```

This matters whenever you read raw memory, interact with binary protocols, or inspect hardware registers.

---

## Alignment and Access Rules

Cortex-M processors have specific alignment rules. Accessing memory at an address that is not aligned to the size of the access can be more expensive or, on some cores, unsupported.

Typical rules are:

- 8-bit access: no special alignment required,
- 16-bit access: address should be even,
- 32-bit access: address should be word-aligned.

On Cortex-M0 and Cortex-M0+, unaligned accesses may not be supported efficiently. On Cortex-M3/M4/M7, unaligned access is often supported by the hardware, but you should still write code that respects alignment for portability and performance.

### Why Alignment

Misaligned access can cause:

- faults,
- slower execution,
- or unexpected behavior on some core variants.

This is especially relevant when reading packed structures from memory or when interpreting data from external peripherals.

---

## Bit-Banding

Some Cortex-M cores support bit-banding. This is a very useful feature that allows software to access a single bit in SRAM or peripheral memory using a special alias region.

The idea is simple:

- you write to a special alias address,
- and the core maps it to a single bit in the original memory region.

This makes single-bit control much easier and can be more efficient than using read-modify-write sequences.

Typical bit-band regions are:

- SRAM bit-band alias region,
- peripheral bit-band alias region.

Example idea:

```c
/* set one bit in a peripheral register */
volatile uint32_t *bit_alias = (volatile uint32_t *)0x22000000;
*bit_alias = 1;
```

The exact addresses depend on the target, but the concept is important because it shows that Cortex-M memory is not just a passive storage space; it is a programmable system with special access mechanisms.

---

## The Memory Protection Unit (MPU)

The Memory Protection Unit is one of the most powerful parts of the Cortex-M memory model. The MPU lets the system define memory regions with permissions and attributes.

A typical MPU region can define:

- base address,
- size,
- access permissions,
- execute permissions,
- shareability,
- cacheability,
- and subregion behavior.

### Why the MPU Matters

The MPU helps prevent:

- code from writing into forbidden regions,
- privileged code from being called incorrectly,
- stack overflows from corrupting adjacent memory,
- and unsafe access to peripheral or flash regions.

In embedded systems, the MPU is often used with RTOSes or safety-oriented firmware to enforce isolation between tasks or between application code and system code.

---

##  Memory Access in Interrupt Handlers

Interrupts add another dimension to the memory model. When an interrupt occurs, the processor saves context onto the stack and switches to handler mode. This means the handler uses the active stack and potentially different privilege state.

That has consequences:

- interrupt handlers must not corrupt the stack used by the interrupted context,
- they must be careful with shared data,
- and they should avoid unbounded recursion or deep local allocations.

In RTOS systems, the memory model is often part of the task context-switch design. The scheduler may switch between `MSP` and `PSP`, or it may use the MPU to isolate task memory regions.

---

## Memory Behavior of Peripherals

Peripheral registers are special because they are not ordinary storage cells. Writing to them may trigger hardware actions, and reading them may return special status information rather than a simple value.

For example:

```c
volatile uint32_t *gpio_odr = (volatile uint32_t *)0x40020014;
*gpio_odr = 0x00000001;
```

This does not “write a value to memory” in the simple software sense. It causes the GPIO hardware to change state.

---

## Common Mistakes Related to the Memory Model

Here are some bugs that often come from misunderstanding the Cortex-M memory model:

### Forgetting `volatile`
A peripheral register read is not like reading a normal variable. If you forget `volatile`, the compiler can optimize away important accesses.

### Writing to the Wrong Memory Region

A buggy pointer can accidentally write into a reserved or privileged region. On Cortex-M, this can turn into a fault or hard fault.

### Stack Overflow
Large local arrays or deep recursion can overflow the stack and corrupt nearby data.

### Misaligned Access
On some cores, misaligned access can cause faults or poor performance.

### Ignoring MPU Configuration
Without proper MPU rules, one task or one code path can overwrite another region in unsafe ways.

---