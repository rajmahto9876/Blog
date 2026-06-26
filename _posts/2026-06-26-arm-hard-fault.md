---
title: "Understanding ARM Cortex-M HardFault"
date: 2026-06-26 14:00:00
categories: arm
tags: [arm, hard-fault, debugging, cortex-m]
---

**Definition:** The HardFault is a fault exception on Cortex-M processors raised for severe error conditions that cannot be handled by other fault handlers (BusFault, UsageFault, MemManage) or when those handlers are disabled or escalate.

**Causes:** 
- Execution of an undefined instruction.
- Access to an invalid memory region without BusFault handling.
- Stack overflows.
- Unaligned accesses (if trapping enabled).
- Divide-by-zero/illegal operations when UsageFault escalation occurs.

**Cortex-M Exception Model**

- Cortex-M has configurable fault handlers: Reset, NMI, HardFault, MemManage (MPU), BusFault, UsageFault, then various system exceptions and interrupts.
- Faults can be _precise_ (information available about the faulting instruction) or _imprecise_. BusFaults from speculative buffering may be imprecise on some cores.
- If a fault handler is disabled or priority rules prevent a more specific handler from running, exceptions can escalate to HardFault.

**Stack frame on exception entry**

When an exception occurs, the CPU pushes a stacked frame onto the active stack (MSP or PSP) containing these registers in this order: `R0`, `R1`, `R2`, `R3`, `R12`, `LR`, `PC`, `xPSR`. If FPU state is present and lazy stacking is used, additional registers are stacked.

`PC` points to the instruction to execute when returning from the exception (often the faulting instruction or the next instruction depending on architecture), and `LR` contains the EXC_RETURN value telling which stack was used and whether floating point state was stacked.

**Key System Registers**

- `CFSR` (Configurable Fault Status Register) — Combine of UsageFault / BusFault / MemManage status bits.
- `HFSR` (HardFault Status Register) — Flags forced HardFaults (HFSR.FORCED) and vector table hard faults.
- `MMFAR` (MemManage Fault Address Register) — Faulting address for MemManage when valid (MMFARVALID bit in CFSR).
- `BFAR` (Bus Fault Address Register) — Faulting address for BusFault when valid (BFARVALID bit in CFSR).

These registers from the fault handler helps in determining the cause. Example bit meanings:

- `CFSR.UsageFault` bits: undefined instruction, invalid state, invalid PC load, unaligned access (UNALIGNED), divide by zero.
- `CFSR.BusFault` bits: instruction/data bus errors, prefetch/postfetch, imprecise data bus error.

**Common Causes**

- Null pointer dereference or invalid function pointer call (PC jumps to 0x0 or garbage).
- Stack overflow: corrupt return addresses on the stack; exception stacking may clobber memory.
- Corrupted vector table or wrong VTOR base — IRQ or exception return lands at invalid address.
- Access to peripheral/memory region not present (mapping mistakes, clocks disabled, wrong bus).
- Unaligned access or access width mismatch when MPU or alignment trapping is enabled.
- Interrupt handler overrunning its stack frame or returning incorrectly.
- Using freed memory or DMA writing into stack/heap.

**Practical Example**

A minimal HardFault handler that captures registers and system fault registers, then loops so a debugger can inspect memory.

```c
// Minimal HardFault handle
#include <stdint.h>

volatile uint32_t hf_r0;
volatile uint32_t hf_r1;
volatile uint32_t hf_r2;
volatile uint32_t hf_r3;
volatile uint32_t hf_r12;
volatile uint32_t hf_lr;
volatile uint32_t hf_pc;
volatile uint32_t hf_psr;

volatile uint32_t hf_cfsr;
volatile uint32_t hf_hfsr;
volatile uint32_t hf_mmfar;
volatile uint32_t hf_bfar;

__attribute__((naked)) void HardFault_Handler(void) {
	__asm volatile (
		"tst lr, #4                \n" // test EXC_RETURN bit to know stack used
		"ite eq                    \n"
		"mrseq r0, msp             \n"
		"mrsne r0, psp             \n"
		"b hard_fault_handler_c    \n"
	);
}

void hard_fault_handler_c(uint32_t *stack_frame) {
	hf_r0  = stack_frame[0];
	hf_r1  = stack_frame[1];
	hf_r2  = stack_frame[2];
	hf_r3  = stack_frame[3];
	hf_r12 = stack_frame[4];
	hf_lr  = stack_frame[5];
	hf_pc  = stack_frame[6];
	hf_psr = stack_frame[7];

	hf_cfsr = (*(volatile uint32_t *)0xE000ED28);
	hf_hfsr = (*(volatile uint32_t *)0xE000ED2C);
	hf_mmfar = (*(volatile uint32_t *)0xE000ED34);
	hf_bfar = (*(volatile uint32_t *)0xE000ED38);

	while (1) {
		__asm volatile("bkpt #0");
	}
}
```

Notes:
- The handler is marked `naked` to get full control over register usage and stack selection.
- `hard_fault_handler_c` reads the stacked registers and system fault registers and then loops.
- Use volatile globals so a debugger can inspect their values without compiler optimization removing them.

**Interpreting the captured data**

- `hf_pc` shows the return address. The faulting instruction is often at `hf_pc - 2` or `hf_pc - 4` depending on thumb 16/32-bit instruction width; disassemble around `hf_pc` to find the precise instruction.
- `hf_lr` contains EXC_RETURN or the link register from the stacked frame — bit patterns in EXC_RETURN indicate which stack was used and whether floating point state was stacked.
- Check `hf_cfsr` and `hf_hfsr`: if `HFSR.FORCED` is set, a configurable fault escalated to HardFault; inspect subfields in CFSR for more details.
- If `MMFARVALID` or `BFARVALID` are set, `hf_mmfar`/`hf_bfar` contain the faulting address.

**Debugger workflow (GDB/OpenOCD or J-Link)**

1. Reproduce the fault with debugger attached; when it hits the `while(1)`.
2. Inspect the volatile saved registers (`hf_pc`, `hf_lr`, `hf_r0` ...).
3. Disassemble around `hf_pc`:
4. If the PC points into flash or RAM where code isn't expected, inspect the vector table and VTOR:
5. Check stack pointer and stack usage to detect overflow:
6. If BFAR/MMFAR are valid, examine the addresses or peripherals at those addresses.


```bash
monitor reset halt
load
continue
# when it stops, in gdb:
info registers
print hf_pc
print/x hf_cfsr

disassemble (hf_pc - 8), (hf_pc + 16)

print/x *(uint32_t*)0xE000ED08  // VTOR

print/x (__get_MSP())
print/x (__get_PSP())
```

**Debugging checklist**

1. Attach debugger and reproduce fault; capture stacked frame.
2. Read `HFSR/CFSR/MMFAR/BFAR` to identify fault class.
3. Disassemble at `hf_pc` and inspect instruction semantics.
4. Verify pointer values used by the instruction are valid and in correct memory regions.
5. Check for stack overflow: compare SP to known stack boundaries and check canary (if used).
6. Inspect vector table and `VTOR` for corruption.
7. Verify MPU configuration and permissions if MPU present.
8. If fault occurs during interrupt, check ISR stack usage, nested interrupts, and priorities.
9. For sporadic/imprecise BusFaults, look for DMA or external agents corrupting memory.
10. Reproduce with compiler optimizations reduced (`-O0` or `-Og`) to make debugging easier.

**Advanced tips**

- Use a stack canary placed at the end of the stack to detect overflow.
- Enable BusFault/MemManage/UsageFault handlers (clear DISABLE bits in `SCB->SHCSR`) early in boot so they trap specifically instead of escalating to HardFault.
- Use linker script to place sentinel values at ends of stacks and periodically check them.
- For hard-to-catch faults, add instrumentation: increment a per-task counter, add assertions, or use a watchdog timer to detect time spent in fault loops.

**Resources and Tools**

- ARMv7-M and ARMv8-M Architecture Reference Manual: exception model and EXC_RETURN behavior.
- CMSIS: `core_cm3.h`/`core_cm4.h` for helper functions to read registers and get SP.
- OpenOCD + GDB, Segger J-Link + J-Link GDB server for live debugging.

