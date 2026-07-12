---
title: "Understanding ARM Cortex-M EXC_RETURN"
date: 2026-07-11 01:00:00
categories: arm
tags: [arm, EXC_RETURN, cortex-m]
---

# Decoding the EXC_RETURN

Understanding the very strange value in the Link Register (`LR` / `R14`) during an interrupt: **`0xFFFFFFF9`** or **`0xFFFFFFFD`**.

The Link Register is supposed to hold the return memory address of the instruction that called the function but it slaso contain a memory address pointing to the absolute end of the 32-bit address space, where no code exists.

---

## How Interrupts Differ from Functions

In a standard C function call, the ARM processor uses the **Link Register (`LR` / `R14`)** to store the return address (`PC`). When the function completes, a simple branch instruction like `BX LR` jumps the Program Counter back to where it left off.

However, handling an **Exception** or an **Interrupt Service Routine (ISR)** is fundamentally different. When a hardware interrupt fires:
1. The processor **save the current context** (the caller-saved registers: `R0-R3`, `R12`, `LR`, `PC`, and `xPSR`) onto the active stack so the interrupted program isn't corrupted.
2. The processor might need to change its **Mode** (from *Thread Mode* to *Handler Mode*).
3. The processor might need to switch its **Stack Pointer** (from the Process Stack Pointer, `PSP`, to the Main Stack Pointer, `MSP`).

If the CPU simply stored the interrupted instruction's address in `LR`, how would it know whether to unstack the registers, switch back to Thread mode, or swap stack pointers when the ISR finishes? 

---

## EXC_RETURN: The Hardware  Word

To solve this, multi-instruction software assembly routines at the end of every C interrupt handler.

When an exception occurs, the hardware automatically stacks the real return address onto the memory stack as part of the context frame. Then, **it loads a special magic token into the Link Register (`LR`) called `EXC_RETURN`**.

This magic token always begins with `0xF` (all 1s in the upper bits). When your ISR finishes and executes a standard return instruction (such as `BX LR` or `POP {PC}`), the ARM core checks the value being loaded into the Program Counter. If it sees a value starting with `0xF...`, it intercepts the branch! Instead of trying to jump to that memory address, the hardware initiates the **Exception Return Sequence**:
1. It reads the specific bits of the `EXC_RETURN` token to determine *how* to return.
2. It automatically pops (`unstacks`) `R0-R3`, `R12`, `LR`, `PC`, and `xPSR` from the correct stack.
3. It restores the processor mode and stack pointer to their pre-interrupt states.

Because of this, **ARM Cortex-M interrupt handlers are written ad  standard C functions!** The compiler simply emits a normal return instruction, and the hardware handles the rest.

---

## EXC_RETURN Bit Layout (ARMv7-M)

The **EXC_RETURN** value is a special constant loaded into the **Link Register (LR)** when an exception is taken. The processor decodes specific bits of this value to determine **how execution should resume** after the exception handler completes.

### EXC_RETURN Format

```text
 31                                               5 4 3 2 1 0
+-------------------------------------------------+-+-+-+-+-+
|                  1 1 1 1 ... 1                  |F|M|S|R|1|
+-------------------------------------------------+-+-+-+-+-+
                                                   | | | | |
                                                   | | | | +-- Bit [0]: Always 1 (ARMv7-M)
                                                   | | | +---- Bit [1]: Reserved (0)
                                                   | | +------ Bit [2]: SPSEL (Stack Pointer Select)
                                                   | +-------- Bit [3]: MODE (Processor Mode)
                                                   +---------- Bit [4]: FTYPE (Frame Type / FPU)
```

### Bit Descriptions

| Bits | Name | Description |
|------|------|-------------|
| **[31:5]** | Prefix | Always `1`. This unique pattern identifies the value as an **EXC_RETURN** code rather than a normal return address. |
| **[4]** | **FTYPE** | Selects the stack frame type. `1` = Standard stack frame (no floating-point registers). `0` = Extended stack frame (includes floating-point context). |
| **[3]** | **MODE** | Determines the processor mode after the exception return. `0` = Return to **Handler mode**. `1` = Return to **Thread mode**. |
| **[2]** | **SPSEL** | Selects which stack pointer to restore. `0` = **Main Stack Pointer (MSP)**. `1` = **Process Stack Pointer (PSP)**. |
| **[1]** | Reserved | Reserved in ARMv7-M and must remain `0`. |
| **[0]** | Fixed | Always `1` for a valid **EXC_RETURN** value. |

## Debugging Tips:

| Scenario | Why EXC_RETURN Matters |
|-----------|------------------------|
| **Debugging HardFaults** | A corrupted or invalid **EXC_RETURN** value is a common cause of HardFaults and UsageFaults. Decoding it quickly reveals which stack pointer was active and where the processor was attempting to return. |
| **RTOS Context Switching** | Real-Time Operating Systems (RTOSes) such as FreeRTOS rely on **EXC_RETURN** to determine whether a task should resume using the **PSP** or **MSP**. Understanding these values makes context switch debugging much easier. |
| **Interrupt Debugging** | During nested interrupts, **EXC_RETURN** indicates whether execution will resume in **Handler mode** or return to the interrupted **Thread mode**, helping trace interrupt flow. |
| **FPU Context Management** | On Cortex-M processors with an FPU, **EXC_RETURN** tells the processor whether an extended floating-point stack frame must be restored, preventing floating-point register corruption. |
| **Bare-Metal Development** | Even without an RTOS, knowing how **EXC_RETURN** works helps diagnose unexpected interrupt behavior, stack corruption, and incorrect exception returns. |
| **Stack Analysis** | By inspecting **EXC_RETURN**, engineers can immediately identify whether the exception originated from the **Main Stack Pointer (MSP)** or the **Process Stack Pointer (PSP)**, significantly simplifying crash analysis. |
| **Low-Level Firmware Design** | Bootloaders, custom schedulers, and exception handlers often manipulate or inspect **EXC_RETURN** directly to implement advanced control flow and context restoration mechanisms. |