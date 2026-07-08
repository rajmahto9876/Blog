---
title: "Understanding ARM Cortex-M xPSR Register"
date: 2026-07-08 01:00:00
categories: arm
tags: [arm, xPSR register, cortex-m]
---

The `xPSR` register is one of the most important. In practice, it is a combined view of three logical status components: the Application Program Status Register (APSR), the Interrupt Program Status Register (IPSR), and the Execution Program Status Register (EPSR). When inspected under a debugger, a single xPSR value is seen, but it carries the condition flags, exception state, and Thumb execution state that drive how the CPU behaves.

## xPSR Register

On Cortex-M, the processor status is split into three parts:

- `APSR`: carries arithmetic and logic flags such as `N, Z, C, V, and Q`.
- `IPSR`: holds the current exception number.
- `EPSR`: holds execution-related state such as the Thumb state and IT-block information.

That is why xPSR is often described as the “program status register” for the core. It is not just a flag register. It tells the CPU:

- what the last comparison or arithmetic result was,
- whether the processor is in thread mode or handling an exception,
- and whether the current instruction stream is executing in Thumb state.

## Importance

For Cortex-M, xPSR is involved in almost everything:

- conditional branches rely on `APSR` flags,
- exception handlers depend on `IPSR` to know which interrupt or fault is active,
- and the core uses `EPSR/T-bit` state to decode instructions correctly.


## xPSR Register Bits:

### 1. APSR: condition flags and arithmetic state

The `APSR` contains the condition flags used by instructions like `CMP`, `ADD`, `SUB`, and `TST`.

The commonly used bits are:

- Bit 31 — N (Negative)
  - 1 means the result was negative.
  - 0 means the result was non-negative.

- Bit 30 — Z (Zero)
  - 1 means the result was zero.
  - 0 means the result was non-zero.

- Bit 29 — C (Carry)
  - 1 means an unsigned carry occurred.
  - 0 means no carry occurred.

- Bit 28 — V (Overflow)
  - 1 means a signed overflow occurred.
  - 0 means no overflow occurred.

- Bit 27 — Q (Saturation)
  - Used with DSP-style saturating instructions.
  - It indicates that saturation occurred.

- Bits 19:16 — GE[3:0]
  - Used with some SIMD and DSP operations.
  - These are less commonly used in everyday embedded code.

A common pattern is:

```c
CMP R0, R1      ; update APSR flags
BNE not_equal   ; branch based on Z flag
```

And in C, the same effect when using comparisons in conditionals.

### 2. IPSR: which exception is running

The `IPSR` holds the current exception number. This is useful when debugging interrupts and faults.

- Value `0` means the processor is in thread mode.
- Values such as `1` for Reset, `2` for NMI, `3` for HardFault, and higher values for interrupts such as SysTick or external interrupts indicate the active exception.

This tells exactly what kind of context it is executing in.

### 3. EPSR: execution state and Thumb context

The `EPSR` contains execution-related information. The important one for Cortex-M is the T-bit, which indicates Thumb state.

- On Cortex-M, the processor is expected to run in Thumb state.
- The T-bit is normally `1` for normal Cortex-M code.
- The EPSR also carries IT-block state used by Thumb-2 conditional execution.

Note:The hardware maintains it automatically in Cortex-M.

## xPSR update

The `xPSR` is updated automatically by the core:

- Arithmetic and logical instructions update APSR flags.
- Exceptions update IPSR when entering an ISR or fault handler.
- The CPU manages EPSR state as instructions are decoded and executed.

## xPSR and exception entry

When an exception occurs, the Cortex-M core automatically pushes a stack frame that includes the registers it must restore later. That frame includes the `PC`, `LR`, and `xPSR`.

A typical exception frame looks like this:

```text
+-------------------+
| r0                |
| r1                |
| r2                |
| r3                |
| r12               |
| lr                |
| pc                |
| xPSR              |
+-------------------+
```
If a hard fault occurs, the saved xPSR tells about what the CPU was doing at the moment of failure.

## Bit-level view with code

Debugger View

### Reading APSR flags

```c
#include <stdint.h>

static inline uint32_t get_apsr(void)
{
    uint32_t val;
    __asm volatile("mrs %0, apsr" : "=r"(val));
    return val;
}

void print_apsr_bits(void) {
    uint32_t apsr = get_apsr();

    printf("N=%u Z=%u C=%u V=%u Q=%u\n",
           (apsr >> 31) & 1u,
           (apsr >> 30) & 1u,
           (apsr >> 29) & 1u,
           (apsr >> 28) & 1u,
           (apsr >> 27) & 1u);
}
```

### Reading IPSR to identify the active exception

```c
#include <stdint.h>

static inline uint32_t get_ipsr(void)
{
    uint32_t val;
    __asm volatile("mrs %0, ipsr" : "=r"(val));
    return val;
}

void print_current_exception(void)
{
    uint32_t ipsr = get_ipsr();

    if (ipsr == 0) {
        printf("Thread mode\n");
    } else {
        printf("Exception number: %lu\n", (unsigned long)ipsr);
    }
}
```

### Reading EPSR to inspect Thumb state

```c
#include <stdint.h>

static inline uint32_t get_epsr(void)
{
    uint32_t val;
    __asm volatile("mrs %0, epsr" : "=r"(val));
    return val;
}

void print_epsr_state(void)
{
    uint32_t epsr = get_epsr();
    printf("Thumb state T-bit = %u\n", (epsr >> 24) & 1u);
}
```

## Example

Consider this:

```c
MOV R0, #5
MOV R1, #3
CMP R0, R1
BHI greater
```

Here is what happens:

- `CMP R0, R1` updates the APSR flags.
- Since `5 > 3`, the C flag is set appropriately and the branch condition is evaluated.
- The CPU uses the updated `APSR` state to decide whether `BHI` is taken.

## xPSR in interrupt handling

When an interrupt arrives, the core saves the current context and enters the handler. During that transition:

- `IPSR` changes to reflect the active interrupt number.
- The processor uses the saved `xPSR` to restore the previous execution state later.
- The handler runs in privileged mode, and the stack frame preserves the flags and execution context.

