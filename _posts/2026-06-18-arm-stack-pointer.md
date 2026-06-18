---
title: ARM (Cortex-M) MSP vs PSP - Stack Pointers
date: 2026-06-18
categories: [arm]
tags: [arm, stack-pointer,cortex-m]
---

One of the distinctive features of ARM Cortex-M architecture is the presence of two separate stack pointers: the **Main Stack Pointer (MSP)** and the **Process Stack Pointer (PSP)**. Unlike simpler processors with a single stack pointer, Cortex-M provides this dual-stack architecture to support advanced features like real-time operating systems (RTOS), privileged and unprivileged modes, and improved system robustness.

## Overview of the Two Stack Pointers

### Main Stack Pointer (MSP)

The **MSP (r13_msp)** is the primary stack pointer used for:
- **Exception handlers** - All exception/interrupt handlers always use MSP
- **Kernel code** - RTOS kernel and privileged code
- **Startup code** - The reset handler initializes MSP first
- **Default stack** - The stack pointer used by default in Cortex-M

**Key characteristics:**
- Always accessible in both privileged and unprivileged modes
- Used for system-critical operations
- Has higher priority in hardware
- Initially loaded from Vector[0] at reset

### Process Stack Pointer (PSP)

The **PSP (r13_psp)** is typically used for:
- **User applications** - Unprivileged application code
- **Task contexts** - Individual task stacks in RTOS systems
- **Optional stack** - Not always used in simple systems

**Key characteristics:**
- Can only be used in unprivileged (thread) mode
- Cannot be accessed directly in privileged mode (must use MSR/MRS instructions)
- Each task/thread can have its own PSP
- Increases system isolation and robustness

## Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│         ARM Cortex-M Stack Architecture        │
└─────────────────────────────────────────────────┘

                    PRIVILEGED MODE
                    ───────────────
    ┌──────────────────────────────────────────┐
    │    Main Stack Pointer (MSP)              │
    │    Used by: Exceptions, Kernel, ISRs    │
    │    R13_MSP always accessible            │
    └──────────────────────────────────────────┘

                 UNPRIVILEGED (THREAD) MODE
                 ──────────────────────────
    ┌──────────────────────────────────────────┐
    │   Process Stack Pointer (PSP)            │
    │   Used by: Application Tasks             │
    │   Only accessible in unprivileged mode   │
    └──────────────────────────────────────────┘

    Exception Entry
    ───────────────
    - Always switches to MSP
    - Saves context on MSP
    - Executes handler in privileged mode

    Exception Exit
    ──────────────
    - Restores context from MSP
    - Returns to previous mode
    - PSP restored if exiting to unprivileged mode
```

## Control Register: CONTROL

The **CONTROL** register determines which stack pointer is active:

```
Bit Layout:
┌─────────────────────────────────────────┐
│ 31-3 │ Bit 2 (FPCA) │ Bit 1 (SPSEL) │ Bit 0 (nPRIV) │
├─────────────────────────────────────────┤
│ RES  │  FP Active   │ SP Selection │ Privilege      │
└─────────────────────────────────────────┘

nPRIV (Bit 0):
  0 = Privileged mode
  1 = Unprivileged mode

SPSEL (Bit 1):
  0 = Use MSP (only available in privileged mode)
  1 = Use PSP (only available in unprivileged mode)

FPCA (Bit 2):
  FPU context active (Cortex-M4F, M7, M33)
```

### Setting CONTROL Register

```c
// Read CONTROL register
uint32_t control = __get_CONTROL();

// Write CONTROL register
__set_CONTROL(control);

// Switch to unprivileged mode with PSP
uint32_t control = __get_CONTROL();
control |= 0x03;  // Set both nPRIV and SPSEL
__set_CONTROL(control);

// Switch back to privileged mode with MSP
uint32_t control = __get_CONTROL();
control &= ~0x03;  // Clear both nPRIV and SPSEL
__set_CONTROL(control);
```

## Stack Pointer Register Access

### Accessing Stack Pointers

In Cortex-M architecture, you cannot directly read/write the PSP like a regular register. Instead, use special instructions:

```c
// ========== MSP Access (R13_MSP) ==========
// Read MSP
uint32_t msp = __get_MSP();

// Write MSP
__set_MSP(new_msp_value);

// ========== PSP Access (R13_PSP) ==========
// Read PSP (only in privileged mode)
uint32_t psp = __get_PSP();

// Write PSP (only in privileged mode)
__set_PSP(new_psp_value);

// ========== CONTROL Register ==========
// Read CONTROL
uint32_t control = __get_CONTROL();

// Write CONTROL
__set_CONTROL(new_control);
```

### Assembly Implementation

```c
// Read MSP
    MRS R0, MSP          ; Move MSP to R0

// Write MSP
    MSR MSP, R0          ; Move R0 to MSP

// Read PSP
    MRS R0, PSP          ; Move PSP to R0

// Write PSP
    MSR PSP, R0          ; Move R0 to PSP

// Read CONTROL
    MRS R0, CONTROL      ; Move CONTROL to R0

// Write CONTROL
    MSR CONTROL, R0      ; Move R0 to CONTROL
    ISB                  ; Instruction Synchronization Barrier
```

## Stack Memory Layout

### Single Stack System (MSP Only)

Typical for simple embedded systems without RTOS:

```
┌─────────────────────────────┐
│   Higher Addresses          │
│   ─────────────────         │
│                             │
│                             │
│   [Stack grows downward]    │
│                             │
│   MSP (Current position)    │
│       ↓                     │
│   [Empty/Available]         │
│                             │
│   ─────────────────         │
│   Lower Addresses           │
└─────────────────────────────┘
```

### Dual Stack System (MSP + PSP)

Typical for RTOS with multiple tasks:

```
┌─────────────────────────────┐
│   Higher Addresses          │
│   ─────────────────         │
│  Exception Stack (MSP)      │
│       ↓                     │
│   [Empty]                   │
│                             │
├─────────────────────────────┤
│                             │
│  Task Stack (PSP Task 1)    │
│       ↓                     │
│   [Empty]                   │
│                             │
├─────────────────────────────┤
│                             │
│  Task Stack (PSP Task 2)    │
│       ↓                     │
│   [Empty]                   │
│                             │
│   ─────────────────         │
│   Lower Addresses           │
└─────────────────────────────┘
```

## When Exceptions Occur

The behavior changes depending on the current execution context:

### Exception Entry (Context Save)

```
Before Exception:
├─ In Privileged Mode with MSP
│  └─ Exception saves context on MSP
│
└─ In Unprivileged Mode with PSP
   └─ Exception saves context on MSP (!)
      └─ CPU automatically switches to MSP
```

```c
// Simplified exception entry flow
void ExceptionEntry(void)
{
    // Save current context
    if (unprivileged_mode)
    {
        // Switch to MSP automatically
        save_context_on_MSP();
    } 
    else
    {
        save_context_on_MSP();
    }
    
    // Enter privileged mode
    enter_privileged_mode();
    
    // Execute exception handler
    execute_handler();
}
```

### Exception Exit (Context Restore)

```
After Exception:
├─ Return value in R0 indicates target mode
│  └─ 0xFFFFFFF9, 0xFFFFFFFD, 0xFFFFFFE1 → Thread mode with MSP
│  └─ 0xFFFFFFE9 → Thread mode with PSP
│
└─ Hardware automatically restores context
   └─ Restores appropriate stack pointer
   └─ Returns to previous privilege mode
```

## Code Examples

### Example 1: Basic MSP vs PSP Setup

```c
#include <stdint.h>
#include <stdio.h>

// Initialize main stack for exception handlers
void Init_MSP(void)
{
    uint32_t msp_value = (uint32_t)&__StackTop;  // From linker script
    __set_MSP(msp_value);
    printf("MSP initialized to: 0x%08X\n", msp_value);
}

// Initialize process stack for a task
void Init_PSP(uint32_t *task_stack_top)
{
    __set_PSP((uint32_t)task_stack_top);
    printf("PSP initialized to: 0x%08X\n", (uint32_t)task_stack_top);
}

// Get current stack pointer (works in both MSP and PSP mode)
uint32_t Get_Current_SP(void)
{
    uint32_t control = __get_CONTROL();
    
    if (control & 0x02)
    {  // SPSEL = 1 (PSP active)
        return __get_PSP();
    }

   return __get_MSP();
}
```

### Example 2: Switching to Unprivileged Mode with PSP

```c
#include <stdint.h>

// Declare task stack
uint32_t task_stack[256];  // 1KB stack for task

// Initialize and switch to unprivileged mode
void Switch_To_Unprivileged_Mode(void)
{
    // Ensure we're in privileged mode first
    if (__get_CONTROL() & 0x01) {
        printf("Error: Already in unprivileged mode!\n");
        return;
    }
    
    // Initialize PSP with task stack
    uint32_t psp_value = (uint32_t)&task_stack[256];  // Stack top
    __set_PSP(psp_value);
    
    // Switch to unprivileged mode with PSP
    uint32_t control = __get_CONTROL();
    control |= 0x03;  // Set nPRIV=1, SPSEL=1
    __set_CONTROL(control);
    
    // ISB to ensure the change takes effect
    __ISB();
    
    printf("Switched to unprivileged mode\n");
    printf("Now using PSP: 0x%08X\n", __get_PSP());
}

// Return to privileged mode (only works if called from handler)
void Switch_To_Privileged_Mode(void)
{
    uint32_t control = __get_CONTROL();
    control &= ~0x03;  // Clear nPRIV and SPSEL
    __set_CONTROL(control);
    __ISB();
    printf("Switched to privileged mode\n");
}
```
### Example 3: Exception Handler Behavior

```c
// Global variable to track stack usage
uint32_t exception_count = 0;

// SysTick exception handler - always uses MSP
void SysTick_Handler(void)
{
    // Always executes in privileged mode
    // Always uses MSP, regardless of what was active before
    uint32_t current_sp = __get_MSP();
    
    exception_count++;
    
    // Read PSP to see what task was running
    uint32_t task_sp = __get_PSP();
    
    // The ISR completes and returns to previous mode/stack
}

// SVC (Supervisor Call) handler - used for system calls
void SVC_Handler(void)
{
    // Called from unprivileged code
    // Executes in privileged mode but can access PSP
    uint32_t psp = __get_PSP();  // Get interrupted task's stack
    
    // Extract SVC number from stacked PC and opcode
    uint32_t *sp = (uint32_t *)psp;
    uint32_t pc = sp[6];  // PC is at offset 6 in stacked frame
    uint8_t *instruction = (uint8_t *)(pc - 2);  // SVC instruction
    uint8_t svc_number = instruction[0];
    
    // Handle system call
    switch (svc_number) {
        case 0x00:
            // System service 0
            break;
        case 0x01:
            // System service 1
            break;
    }
}
```

## Stack Frame on Exception Entry

When an exception occurs, the CPU automatically saves this context:

```
Before Exception:          After Exception Entry (saved on MSP):
──────────────             ───────────────────────────────────
  R0-R3                      sp[0] = R0
  R12                        sp[1] = R1
  LR                         sp[2] = R2
  PC                         sp[3] = R3
  PSR                        sp[4] = R12
                             sp[5] = LR (return address)
                             sp[6] = PC
                             sp[7] = xPSR (exception PSR)

Stack Frame Size: 32 bytes (8 * 4-byte words)
```

## Common Mistakes and Solutions

| Mistake | Problem | Solution |
|---------|---------|----------|
| Not initializing MSP | Stack corruption on first exception | Set MSP in reset handler before any exception can occur |
| Using PSP in privileged mode context | Illegal operation or undefined behavior | Only use PSP in unprivileged mode; use MSR/MRS to access it |
| Not synchronizing after CONTROL change | CPU may not recognize mode change | Use ISB (Instruction Synchronization Barrier) after writing CONTROL |
| Insufficient stack size | Stack overflow into other memory | Allocate generous stack sizes; use stack guards in RTOS |
| Exception overwriting task data | Task corruption due to shared stack | Always use separate MSP and PSP when using tasks |

## Common Practices

1. **Always Initialize MSP First**
   ```c
   void Reset_Handler(void)
   {
       __set_MSP((uint32_t)&__StackTop);  // Do this first!
       // ... rest of initialization
   }
   ```

2. **Use ISB After Mode Changes**
   ```c
   __set_CONTROL(control_value);
   __ISB();  // Ensure change takes effect
   ```

3. **Check Stack Alignment**
   ```c
   // Stack pointers must be 8-byte aligned (or 4-byte minimum)
   if (__get_MSP() & 0x07) {
       // Error: misaligned stack!
   }
   ```

4. **Protect Privileged Operations**
   ```c
   // Only set PSP from privileged mode
   if ((__get_CONTROL() & 0x01) == 0) {  // If privileged
       __set_PSP(new_psp);
   }
   ```

5. **Document Stack Layout in Linker Script**
   ```text
   .msp_stack :
   {
       . += 0x1000;  // 4KB MSP stack
       __StackTop = .;
   } > RAM
   
   .psp_stacks :
   {
       __PSP_TaskStack_0 = .;
       . += 0x0800;  // 2KB per task
       __PSP_TaskStack_1 = .;
       . += 0x0800;
   } > RAM
   ```

## References

| Aspect | MSP | PSP |
|--------|-----|-----|
| **Usage** | Exceptions, kernel, ISRs | Application tasks |
| **Mode** | Privileged only | Unprivileged |
| **Access** | Direct via R13_MSP | Via MRS/MSR instructions |
| **Automatic Switch** | On exception entry | Manual in RTOS |
| **CONTROL Bit** | SPSEL=0 | SPSEL=1 |
| **Default** | Used at startup | Initialized by RTOS |

- ARM Cortex-M Architecture Reference Manual
- ARM Cortex-M Programming Guide
- CMSIS-CORE Documentation
- RTOS Implementation Guides
