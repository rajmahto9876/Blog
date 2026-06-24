---
title: Understanding PendSV (Cortex-M)
date: 2026-06-24
categories: [arm]
tags: [arm, PendSV, cortex-m]
---

PendSV (Pending Supervisor Call) is a Cortex-M exception designed specifically for context switching and deferred kernel work. In ARM Cortex-M systems, PendSV provides a safe, low-priority entry point for switching tasks, saving and restoring processor state without disturbing higher-priority interrupts.

## Why PendSV Exists

Cortex-M cores use a nested interrupt model. Exceptions higher than PendSV are handled first, so using PendSV for task switching means the switch happens only when the processor is otherwise idle. This avoids disrupting time-critical ISR execution and keeps context switching deterministic.

PendSV is typically triggered by the scheduler from one of these sources:

- SysTick timer for periodic scheduling
- PendSV exception pending bit set by software
- A system call or supervisor call that requests a task switch

## Cortex-M Exception Priorities and PendSV

On Cortex-M, exceptions have configurable priority values. PendSV is normally assigned the lowest priority so all interrupts can complete before the scheduler takes over. This is important because PendSV must run after the state of the interrupted task is stable and after any interrupt-driven events are finished.

## Cortex-M Stacking Behavior

The Cortex-M core automatically saves a basic exception stack frame when an exception occurs. The hardware stack frame contains:

- R0, R1, R2, R3
- R12
- LR (R14)
- PC (R15)
- xPSR

This automatic stacking happens on the active stack: Main Stack Pointer (MSP) during exceptions and Process Stack Pointer (PSP) during thread mode if configured.

A full context switch requires saving additional registers beyond the hardware-saved frame:

- R4-R11
- Optionally other processor state like floating-point registers if the FPU is in use

## Basic Cortex-M Context Switch Sequence

A context switch using PendSV usually follows these steps:

1. SysTick or another interrupt decides a task switch is needed.
2. The scheduler updates the ready task list and chooses the next thread.
3. Software sets the `PENDSVSET` bit in the `ICSR` register (`SCB->ICSR |= SCB_ICSR_PENDSVSET_Msk`).
4. The current ISR returns.
5. The processor enters the PendSV handler at the lowest priority.
6. PendSV handler saves the current task context and restores the next task context.
7. PendSV returns, and execution resumes in the newly restored task.

## PendSV Handler Details

The PendSV handler is the heart of the context switch. It typically performs these operations:

- Save registers R4-R11 of the outgoing task
- Save the current PSP into the task control block (TCB)
- Load the PSP of the next task from its TCB
- Restore registers R4-R11 for the incoming task
- Return from exception using `BX LR`

The handler uses the Process Stack Pointer (PSP) because each thread maintains its own stack. The Main Stack Pointer (MSP) remains reserved for exceptions and the kernel.

## PendSV Handler Workflow

### Saving the current task

The handler must store the current task's context before switching away. On Cortex-M, the handler runs in handler mode on the MSP, but it can switch to the PSP of the current task to save the full thread context.

### Restoring the next task

After selecting the next ready task, PendSV restores its saved PSP and register set. Once done, a return from exception restores the hardware stack frame and resumes the next thread.

## Stack Layout for Context Switching

A typical task stack layout after a context switch looks like this:

| Stack Entry | Register | Description |
|---|---|---|
| Top | xPSR | Program status word containing condition flags and execution state. |
| | PC | Return address for the task when it resumes. |
| | LR | Link register for exception return and subroutine return path. |
| | R12 | Intra-procedure-call scratch register. |
| | R3 | General-purpose scratch register, saved by hardware. |
| | R2 | General-purpose scratch register, saved by hardware. |
| | R1 | General-purpose scratch register, saved by hardware. |
| | R0 | General-purpose register also used for function arguments and return values. |
| | R11 | Callee-saved register saved by software during context switch. |
| | R10 | Callee-saved register saved by software during context switch. |
| | R9 | Callee-saved register saved by software during context switch. |
| | R8 | Callee-saved register saved by software during context switch. |
| | R7 | Callee-saved register saved by software during context switch. |
| | R6 | Callee-saved register saved by software during context switch. |
| | R5 | Callee-saved register saved by software during context switch. |
| Bottom | R4 | Callee-saved register saved by software during context switch. |

The hardware pushes the first eight registers automatically when entering the exception. The PendSV handler pushes R4-R11 manually, so the saved stack contains the full context needed for task restore.

## Implementation Notes

### 1. Use PSP for threads

Each user task should use the Process Stack Pointer. The kernel and exceptions run on MSP. This separation prevents user threads from tampering with kernel stacks and simplifies the switch logic.

### 2. Keep PendSV priority lowest

Set the PendSV priority to the lowest available value. This ensures that PendSV executes only after all interrupt handlers have completed.

### 3. Mask interrupts during switch updates

The scheduler should disable interrupts or set a BASEPRI level when modifying shared scheduling structures. However, the actual context restore is done in PendSV at the lowest priority.

## Example Register Save/Restore Pattern

In assembly, the standard pattern looks like this:

- `MRS R0, PSP`          ; read current thread stack pointer
- `STMDB R0!, {R4-R11}`  ; push R4-R11 onto thread stack
- store R0 in current TCB
- load next task PSP from TCB into R0
- `LDMIA R0!, {R4-R11}`  ; restore R4-R11 for next task
- `MSR PSP, R0`          ; write next thread stack pointer
- `BX LR`                ; return from PendSV

This pattern preserves the thread state across task switches.

## Using PendSV with SysTick

A common scheduler uses SysTick as the periodic tick source. The SysTick handler does not change context directly. Instead it:

- increments the system tick counter
- updates task timing or ready lists
- decides if a context switch is required
- sets `SCB->ICSR |= SCB_ICSR_PENDSVSET_Msk`

By deferring the switch to PendSV, the SysTick handler remains short and predictable.

## Context Switching on Cortex-M Without an OS

Even in bare-metal systems, PendSV can be used to implement cooperative or preemptive multitasking. The basic components are:

- task control block (TCB) with saved PSP
- stack allocation per task
- task creation routine that initializes the stack frame
- scheduler that chooses the next task
- SysTick or event source to trigger PendSV
- PendSV handler to switch tasks

This approach gives a lightweight, efficient task switch mechanism without a full RTOS.
