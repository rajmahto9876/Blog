---
title: Understanding ARM Registers
date: 2026-05-01
categories: [ARM]
tags: [arm, assembly, registers]
---

## Introduction to ARM Registers
Registers hold the data and instructions that the processor is currently working on. Understanding how the ARM architecture utilizes registers is important for ARM.

The ARM architecture is a Load-Store architecture. This means data must be moved from memory into a register before it can be processed. Once processed, the result is stored back into memory from the register.

The 32-bit ARM architecture comprises of the following.

![ARM 32-bit Registers](/assets/image/arm-32-bit-register.jpg)

## General-Purpose Registers (R0 - R12)

The 32-bit ARM architecture features 16 visible registers (R0 through R15). The first 13 of these, **R0 to R12**, are considered general-purpose registers. They are used to hold data, addresses, or intermediate results during computation.

* **R0 - R3:** These are often used as "scratch" registers. According to the ARM Procedure Call Standard (AAPCS), they are used to pass arguments into functions and return values from functions. They are volatile, meaning a function can overwrite them without needing to restore their original values.
* **R4 - R11:** These are typically used for local variables within a function. They are non-volatile (or callee-saved), meaning if a function wants to use them, it must first save their original values (usually to the stack) and restore them before returning.
* **R12:** Often known as the Intra-Procedure-call scratch register (IP), used by the linker as a scratch register.

## Special-Purpose Registers (R13 - R15)

The last three registers have specific hardware-defined.
### R13: Stack Pointer (SP)
The Stack Pointer holds the memory address of the top of the stack. The stack is a vital region of memory used for temporarily storing data, such as local variables, and preserving register values when calling functions. When you push data onto the stack, the SP decreases (ARM stacks typically grow downwards in memory), and when you pop data off, the SP increases.

### R14: Link Register (LR)
The Link Register holds the return address when a function call is made. When a `BL` (Branch with Link) instruction is executed to jump to a subroutine, the address of the next instruction (the one immediately following the `BL`) is automatically saved in the LR. When the subroutine finishes, it simply copies the LR back into the Program Counter (PC) to return control to the caller.

### R15: Program Counter (PC)
The Program Counter is arguably the most critical register. It holds the address of the *next* instruction the CPU will execute. Modifying the PC directly causes the processor to branch to a different part of the code. In ARM state, the PC is typically 8 bytes ahead of the currently executing instruction due to the instruction pipeline.

## Current Program Status Register (CPSR)

Alongside R0-R15, there is a special register that tracks the current state of the processor: the **CPSR (Current Program Status Register)**. 

The CPSR contains:
* **Condition Flags (N, Z, C, V):** These flags are updated by arithmetic and logical operations.
    * **N (Negative):** Set if the result is negative.
    * **Z (Zero):** Set if the result is zero.
    * **C (Carry):** Set if an operation caused a carry out.
    * **V (Overflow):** Set if an operation caused a signed overflow.
* **Control Bits:** These determine the processor's operating mode (e.g., User, Supervisor, IRQ) and state (ARM or Thumb mode), as well as interrupt mask bits.

## A Note on 64-bit ARM (AArch64)

The 64-bit ARM architecture (AArch64) significantly expands the register set. It features 31 general-purpose 64-bit registers (**X0 to X30**), with the lower 32 bits accessible as **W0 to W30**. It also introduces a dedicated zero register (`XZR`/`WZR`) and separates the Program Counter (`PC`) and Stack Pointer (`SP`) from the general-purpose register pool.