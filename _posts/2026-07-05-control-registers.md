---
title: "Understanding ARM Cortex-M Control Register"
date: 2026-07-05 01:00:00
categories: arm
tags: [arm, Control register, cortex-m]
---

The `CONTROL` register shapes thread-mode privilege and stack-pointer selection, and  when a Floating Point Unit (FPU) is present, indicates floating-point context activity. 


## CONTROL Register (conceptual)

Cortex-M distinguishes two execution modes: *handler* (exception) mode and *thread* mode. Handlers always run privileged and use the Main Stack Pointer (`MSP`). Thread mode may run privileged or unprivileged and may use either `MSP` or Process Stack Pointer (`PSP`). The `CONTROL` register selects those thread-mode properties.

- Switching `SPSEL` changes which stack pointer the core uses in thread mode.
- Setting `nPRIV = 1` drops privileges in thread mode (you cannot raise privileges from unprivileged thread mode by writing `CONTROL`).
- `FPCA` flags whether the floating-point state is active for lazy stacking and context save/restore.

These choices affect security boundaries, exception entry/exit behavior, and RTOS perform context switches.

## Bit-level Explanation(Imp Registers):

Following are the bit values of Imp registers:
- Bit 0 — `nPRIV` (Thread mode privilege)
	- 0: Thread mode runs privileged
	- 1: Thread mode runs unprivileged
	- Notes: Attempting to clear `nPRIV` while executing in unprivileged thread mode is ignored. Only privileged software (e.g., an SVC handler or boot code) can restore privilege.

- Bit 1 — `SPSEL` (Stack pointer selection)
	- 0: Use `MSP` for thread mode
	- 1: Use `PSP` for thread mode
	- Notes: The stack pointers themselves are independent registers; you must initialize `PSP` before switching to it.

- Bit 2 — `FPCA` (Floating-point context active) — optional
	- Present only when an FPU or floating-point state is implemented.
	- Indicates whether the floating-point context is active for the current thread. It affects lazy FPU stacking and whether the FPU registers are saved on exception entry.

Higher bits are either reserved or implementation-defined specific to cortex-M series cores.

## Behavior differences: Thread vs Handler mode

- Handler mode: Always privileged; `MSP` is always used.
- Thread mode: `CONTROL` determines privilege and which SP is used.

Exception entry always switches to handler mode and `MSP`. When returning from an exception, the return value encodes whether thread mode should use `MSP` or `PSP` and whether it runs privileged or unprivileged.

Because handlers are privileged, a common pattern is:

- Use `MSP` for interrupts and kernel/OS code.
- Use `PSP` for application threads/tasks (each task has its own `PSP`).

## Accessing `CONTROL` (instructions and CMSIS)

At the instruction level read and write `CONTROL` with `MRS` / `MSR`. After writing `CONTROL`, execute an `ISB` to ensure the change takes effect before subsequent instructions:

```c
		MRS R0, CONTROL    ; read CONTROL
		ORR R0, R0, #2     ; set SPSEL -> use PSP in thread mode
		MSR CONTROL, R0    ; write back
		ISB                ; ensure pipeline uses the new settings
```

Using CMSIS helpers:

```c
#include "cmsis_gcc.h" // or core_cm*.h depending on your SDK

uint32_t ctrl = __get_CONTROL();
ctrl |= 2;            // set SPSEL to PSP
__set_CONTROL(ctrl);
__ISB();              // ensure effect
```

Note: unprivileged code cannot set `nPRIV` to 0 (regain privilege). Writes that would raise privilege are ignored when executed at unprivileged level.

## Common patterns

- Boot/Reset: core resets to `MSP` and privileged thread mode (or handler/boot flow). Boot code typically initializes stacks and sets up `PSP` values for tasks before switching.
- RTOS task switch: kernel runs on `MSP` (privileged), sets `PSP` to the task stack, populates threads' initial stack frame, sets `SPSEL=1`, and drops `nPRIV` (optional) so tasks run unprivileged.
- User-mode sandboxing: software drops `nPRIV` to reduce damage from buggy or malicious application code.
- FPU-aware switching: if core has an FPU, the OS must follow the FP lazy-save semantics; `FPCA` indicates an active FP context.

## Example: switch to PSP and unprivileged thread mode

This minimal sequence is typical for moving from boot (privileged/MSP) into a user task (PSP, unprivileged):

```c
/* 1) Prepare PSP for the new thread */
__set_PSP(task_stack_top);

/* 2) Select PSP for thread mode */
uint32_t ctrl = __get_CONTROL();
ctrl |= (1U << 1);   // set SPSEL = 1 -> PSP
__set_CONTROL(ctrl);
__ISB();             // synchronize

/* 3) Optionally drop privileges for the thread */
ctrl = __get_CONTROL();
ctrl |= 1U;          // set nPRIV = 1 -> unprivileged thread
__set_CONTROL(ctrl);
__ISB();

/* Now branch to thread entry; thread uses PSP and runs unprivileged */
thread_entry();
```

Notes:
- Initialize `PSP` before selecting it; otherwise the core will fetch the wrong stack pointer.
- Use `ISB()` after `MSR CONTROL` (or `__set_CONTROL`) to flush the pipeline and ensure subsequent execution honors the new permissions and SP selection.

## Interaction with other control registers

- `PRIMASK`: globally masks (disables) interrupts (except faults). It does not change privilege or stack selection.
- `BASEPRI`: sets a priority threshold for exception masking. It doesn't alter `CONTROL` functionality, but combined use is common in ISR/critical-section design.
- `FAULTMASK`: masks all exceptions (including configurable faults) and is privileged-only to modify.

Privilege and masking are orthogonal: you may be unprivileged (`CONTROL.nPRIV=1`) and still possess interrupts enabled/disabled via `PRIMASK` set by a privileged kernel earlier.

## Debugging tips:

- Use a debugger to inspect `CONTROL` and the two stack pointers `MSP` and `PSP`.
- Single-step around `MSR CONTROL` and put the watch point on the active stack pointer after `ISB` to verify `SPSEL` changes.

---

