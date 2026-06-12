---
title: Understanding ARM (Cortex-M) Exception Model
date: 2026-06-12
categories: [arm]
tags: [arm, exception, interrupt, cortex-m]
---

The **ARM Cortex-M exception model** is a unified interrupt and exception handling mechanism that provides a consistent, priority-based approach to managing both processor exceptions and external interrupts.

## What is an Exception?
An **exception** is an event that disrupts normal program execution flow. In ARM Cortex-M processors, exceptions include:
- **System exceptions**: NMI, HardFault, SVCall, PendSV, SysTick, etc.
- **External interrupts**: From peripherals managed by NVIC

## Exception Types and Numbers

ARM Cortex-M defines exceptions using a numbering system where each exception has a unique number for its handling.

### System Exceptions (Exception Numbers 1-15)

| Number | Name | Type | Priority |
|--------|------|------|----------|
| 1 | Reset | Non-maskable | -3 (highest) |
| 2 | NMI | Non-maskable | -2 |
| 3 | HardFault | Fault | -1 |
| 4 | MemManage | Fault | Programmable |
| 5 | BusFault | Fault | Programmable |
| 6 | UsageFault | Fault | Programmable |
| 7-10 | Reserved | — | — |
| 11 | SVCall | Service Call | Programmable |
| 12 | DebugMonitor | Debug | Programmable |
| 13 | Reserved | — | — |
| 14 | PendSV | Service Call | Programmable |
| 15 | SysTick | Timer | Programmable |
| 16+ | External IRQs | Interrupt | Programmable |

### Exception Number to IRQ Conversion

```c
// Convert exception number to IRQ number
// IRQ 0 corresponds to exception 16
IRQn_Type irq = exception_number - 16;  // For external interrupts
```

## Exception Priority

Exception priority determines which exception is serviced first when multiple exceptions occur simultaneously.

### Fixed Priority Exceptions

```
Priority -3: Reset (highest, can't change)
Priority -2: NMI
Priority -1: HardFault
```

These three exceptions have fixed, immutable priorities.

### Programmable Priority Exceptions

```c
// Other exceptions can have programmable priority
// Lower numeric value = Higher priority (will preempt lower priority)

#define PRIORITY_SYSTICK    5
#define PRIORITY_UART       10
#define PRIORITY_GPIO       15

NVIC_SetPriority(SysTick_IRQn, PRIORITY_SYSTICK);
NVIC_SetPriority(UART0_IRQn, PRIORITY_UART);
NVIC_SetPriority(GPIO_IRQn, PRIORITY_GPIO);
```

## Exception States

Each exception can be in one of several states:

### 1. **Inactive**
- Exception not pending or active
- No request has been made

### 2. **Pending**
- Exception request has been received
- Waiting to be handled
- Can be pending while another exception is active

### 3. **Active**
- Exception handler is currently executing
- Processor is running in the exception handler

### 4. **Active and Pending**
- Exception handler is executing
- Same exception has been triggered again
- Will restart after current handler completes

State transitions:
```
Inactive → Pending → Active → Inactive
            ↓         ↓
          (stays pending while active)
```

## Vector Table

The **vector table** is an array of exception handler addresses stored at the beginning of memory. The processor uses this table to determine which function to execute when an exception occurs.

### Default Vector Table Location

- **Cortex-M0/M0+**: 0x00000000 (usually in Flash)
- **Cortex-M3/M4/M7**: Configurable via VTOR (Vector Table Offset Register)

### Vector Table Structure

```c
// Typical vector table layout
typedef struct {
    uint32_t stack_pointer;           // Exception 0 (not an exception)
    void (*reset_handler)(void);      // Exception 1
    void (*nmi_handler)(void);        // Exception 2
    void (*hardfault_handler)(void);  // Exception 3
    // ... more exception handlers ...
    void (*systick_handler)(void);    // Exception 15
    // External interrupt handlers start here (Exception 16+)
    void (*irq_handlers[240])(void);  // Up to 240 external interrupts
} VectorTable_t;
```

### Relocating Vector Table

```c
// Move vector table to a different location (usually RAM)
#define VECTOR_TABLE_OFFSET 0x20000000  // RAM address

__asm void RelocateVectorTable(void) {
    ldr r0, =VECTOR_TABLE_OFFSET
    ldr r1, =(0xE000ED08)  // VTOR register address
    str r0, [r1]
    bx lr
}
```

## Exception Stack Frame

When an exception occurs, the processor automatically saves context (registers) on the stack. This is called the **exception stack frame**.

### Stack Frame Structure

```
SP (before exception) ──→ [xPSR]      ← Processor Status Register
                         [PC]        ← Program Counter (return address)
                         [LR]        ← Link Register (return address alternate)
                         [R12]       ← R12 register
                         [R3]        ← R3 register
                         [R2]        ← R2 register
                         [R1]        ← R1 register
                         [R0]        ← R0 register
```

The processor automatically pushes these 8 registers (32 bytes total) onto the stack.

### Extended Stack Frame (FPU)

For Cortex-M4/M7 with FPU enabled, additional floating-point registers are saved:

```
SP + 32  ──→ [S15]
            [S14]
            [S13]
            [S12]
            [S11]
            [S10]
            [S9]
            [S8]
            [FPSCR]
```

## Exception Entry

When an exception occurs, the processor:

1. **Completes current instruction** (mostly)
2. **Saves registers** to stack (8-10 registers)
3. **Clears lower bits of LR** and sets special value
4. **Loads PC from vector table**
5. **Enters exception handler**

## Exception Return

The processor uses special return addresses stored in the **Link Register (LR)** to determine what happens when an exception handler exits.

### Special LR Values

| LR Value | Meaning | Action |
|----------|---------|--------|
| 0xFFFFFFF1 | Return to Handler mode, MSP | Stay in exception handler |
| 0xFFFFFFF9 | Return to Thread mode, MSP | Return to main code (MSP) |
| 0xFFFFFFFD | Return to Thread mode, PSP | Return to main code (PSP) |
| 0xFFFFFFE1 | Return to Handler, MSP (FPU) | Handler with FPU context |
| 0xFFFFFFE9 | Return to Thread, MSP (FPU) | Thread with FPU context |

### Returning from Exception

```c
void MyException_Handler(void)
{
    // Process exception
    
    // Return instruction
    // The processor automatically uses LR to determine return
}
```

## Exception Classes

### 1. **Reset Exception**

```c
void Reset_Handler(void)
{
    // Initialize stack pointer
    // Call main()
    main();
}
```

### 2. **NMI (Non-Maskable Interrupt)**

Cannot be disabled; highest priority after Reset.

```c
void NMI_Handler(void)
{
    // Handle critical condition
    // cannot be masked
}
```

### 3. **Hard Fault Exception**

Catch-all for various fault conditions.

```c
void HardFault_Handler(void)
{dlers
    // indicates a serious error
    while(1);  // Halt
}
```

### 4. **Fault Exceptions (M3+)**

More specific fault handling:

```c
void MemManage_Handler(void)
{
    // Memory protection fault
}

void BusFault_Handler(void)
{
    // Bus error
}

void UsageFault_Handler(void)
{
    // Invalid instruction or operation
}
```

### 5. **System Service Call (SVCall)**

Used for system calls in protected environments.

```c
void SVC_Handler(void)
{
    // Handle supervisor call
    // Extracted SVC number from instruction
}
```

### 6. **PendSV (Pendable Service Call)**

Used in context switching, RTOS schedulers.

```c
void PendSV_Handler(void)
{
    // Perform context switch
}
```

### 7. **SysTick Exception**

System timer interrupt for OS tick.

```c
void SysTick_Handler(void)
{
    // Called at regular intervals
    ticks++;
}
```

## Practicals Examples

### Basic Interrupt Handler

```c
#include "core_cm4.h"

// Handler for UART0 interrupt
void UART0_IRQHandler(void)
{
    uint32_t status = UART0->SR;
    
    if (status & UART_SR_RXNE)
    {
        uint8_t data = UART0->DR;
        uart_receive_buffer[rx_index++] = data;
    }
    
    if (status & UART_SR_TXE)
    {
        if (tx_index < tx_count)
        {
            UART0->DR = uart_transmit_buffer[tx_index++];
        }
    }
}
```

### Protected Interrupt Context

```c
void critical_section(void)
{
    // Save current priority
    uint32_t primask = __get_PRIMASK();
    
    // Disable all interrupts
    __disable_irq();
    
    // Critical code
    shared_variable++;
    
    // Restore interrupt state
    __set_PRIMASK(primask);
}
```

### Priority-Based Nesting

```c
// High priority - can interrupt others
void Timer_Handler(void)
{  // Priority 2
}

// Lower priority - can be interrupted
void GPIO_Handler(void)
{   // Priority 8
}
```

### Exception Information

```c
// Get current exception being handled
uint32_t get_current_exception(void)
{
    uint32_t icsr = SCB->ICSR;
    uint32_t exception_num = icsr & SCB_ICSR_VECTACTIVE_Msk;
    return exception_num;
}

// Check if any exception is active
uint32_t is_exception_active(void)
{
    uint32_t icsr = SCB->ICSR;
    return (icsr & SCB_ICSR_VECTACTIVE_Msk) != 0;
}
```

## Exception Configuration

### Configuring Exception Priority Groups

```c
// Set priority grouping - determines preemption capability
// Different values create different preemption/subpriority splits
NVIC_SetPriorityGrouping(3);  // 8 preemption levels, 2 subpriority
```

### Enabling/Disabling Exceptions

```c
// Enable usage fault
SCB->SHCSR |= SCB_SHCSR_USGFAULTENA_Msk;

// Enable bus fault
SCB->SHCSR |= SCB_SHCSR_BUSFAULTENA_Msk;

// Enable memory management fault
SCB->SHCSR |= SCB_SHCSR_MEMFAULTENA_Msk;
```

## Exception Latency

**Exception latency** is the time from exception trigger to exception handler execution.

### Components of Latency

1. **Recognition delay**: ~1 cycle (exception detected)
2. **Stacking time**: ~12-24 cycles (registers saved to stack)
3. **Handler startup**: ~1-2 cycles (first instruction executed)

**Total typical latency**: 12-25 cycles

## Common Exception Scenarios

### Scenario 1: Timer Interrupt with High Priority

```c
void SysTick_Handler(void)
{
    // Runs every 1ms (priority 1)
    milliseconds++;
}

void GPIO_Handler(void)
{
    // Can be interrupted by SysTick (priority 5)
    // Do GPIO processing
}
```

### Scenario 2: Handling Multiple Interrupts

```c
void UART0_Handler(void)
{  // Priority 3
    rx_data = UART0->DR;
}

void UART1_Handler(void)
{  // Priority 4
    rx_data = UART1->DR;
    // UART0 handler can interrupt this
}

void SPI_Handler(void)
{    // Priority 8
    // Both UART handlers can interrupt this
    spi_data = SPI->DR;
}
```

## Debug Exception Register

```c
// Check for pending exceptions
uint32_t pending_icsr = SCB->ICSR;

// Extract pending exception number
uint32_t pending_exception = (pending_icsr >> 12) & 0xFF;

// Check if HardFault is active
uint32_t hardfault_active = SCB->ICSR & SCB_ICSR_VECTACTIVE_Msk;
```

## References
- ARM Cortex-M3/M4 Technical Reference Manual
- ARM M-Profile Architecture Reference Manual
- CMSIS (Cortex Microcontroller Software Interface Standard) Documentation
