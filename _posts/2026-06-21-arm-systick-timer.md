---
title: Understanding SYSTICK Timer (Cortex-M)
date: 2026-06-21
categories: [arm]
tags: [arm, systick timer, cortex-m,]
---

The SysTick timer (System Tick Timer) is a critical component of all ARM Cortex-M processors. It's a simple, 24-bit countdown timer that is deeply integrated into the processor core and serves as the foundation for real-time operating systems (RTOS), task scheduling, and time-based operations. 

## What is SysTick?

**SysTick** is a system timer that:
- Provides a periodic interrupt mechanism
- Is integrated directly into the Cortex-M processor core
- Operates at a known, configurable frequency
- Is used primarily for scheduling, timing, and context switching in RTOS environments
- Has a fixed 24-bit resolution
- Can be clocked from either the processor clock or an external reference clock

The SysTick timer is one of the few resources that is guaranteed to be present in every Cortex-M processor (ARMv6-M, ARMv7-M, ARMv8-M, etc.), making it a reliable choice for implementing time-based services.

## SysTick Architecture Overview

### Block Diagram

```
┌─────────────────────────────────────────────┐
│         Cortex-M SysTick Block              │
├─────────────────────────────────────────────┤
│                                             │
│  Clock Input → ┌──────────────┐            │
│                │  Prescaler   │            │
│                └──────┬───────┘            │
│                       │                    │
│                       ▼                    │
│              ┌──────────────────┐          │
│              │ 24-bit Decrement │          │
│              │    Counter       │          │
│              └──────┬───────────┘          │
│                     │                      │
│         ┌───────────┴──────────┐           │
│         │ Reload (if enabled)  │           │
│         └───────────┬──────────┘           │
│                     │                      │
│         ┌───────────▼──────────┐           │
│         │  Interrupt Control   │           │
│         │  (SysTick_Handler)   │           │
│         └──────────────────────┘           │
│                                             │
└─────────────────────────────────────────────┘
```

### Key Characteristics

| Feature | Details |
|---------|---------|
| **Counter Width** | 24-bit (0 to 16,777,215) |
| **Reload Capability** | Auto-reload on underflow |
| **Interrupt** | Generates SysTick interrupt when counter reaches 0 |
| **Memory-mapped Registers** | Located at fixed addresses in SCS |
| **Clock Sources** | Processor clock or external reference |
| **Power States** | Available in most power modes |

## SysTick Registers

The SysTick timer is controlled through four memory-mapped registers in the System Control Space (SCS) at addresses `0xE000E000 - 0xE000E010`.

### 1. SysTick Control and Status Register (SYST_CSR)

**Address:** `0xE000E010`

```c
typedef struct
{
    uint32_t ENABLE    : 1;    // Bit 0: Counter enable
    uint32_t TICKINT   : 1;    // Bit 1: SysTick exception request enable
    uint32_t CLKSOURCE : 1;    // Bit 2: Clock source selection
    uint32_t          : 13;    // Bits 3-15: Reserved
    uint32_t COUNTFLAG : 1;    // Bit 16: Count flag (read-only)
    uint32_t          : 15;    // Bits 17-31: Reserved
} SYST_CSR_t;
```

**Bit Descriptions:**
- **ENABLE (0):** Enables the counter (0 = disabled, 1 = enabled)
- **TICKINT (1):** If 1, enables SysTick exception when counter reaches 0
- **CLKSOURCE (2):** 0 = External clock, 1 = Processor clock
- **COUNTFLAG (16):** Set to 1 when counter has counted to 0 since last read (automatically cleared on read)

### 2. SysTick Reload Value Register (SYST_RVR)

**Address:** `0xE000E014`

```c
typedef struct
{
    uint32_t RELOAD : 24;      // Bits 0-23: Reload value (0 to 16,777,215)
    uint32_t       : 8;        // Bits 24-31: Reserved
} SYST_RVR_t;
```

**Usage:** Specifies the value to load into the counter when it reaches 0. This creates the periodic timing.

**Calculation:**
```
Reload_Value = (Desired_Period_ms × Clock_Frequency_MHz) - 1
```

### 3. SysTick Current Value Register (SYST_CVR)

**Address:** `0xE000E018`

```c
typedef struct
{
    uint32_t CURRENT : 24;     // Bits 0-23: Current counter value
    uint32_t        : 8;       // Bits 24-31: Reserved
} SYST_CVR_t;
```

**Usage:** Holds the current value of the counter. Writing to this register clears both the counter and the COUNTFLAG bit in SYST_CSR.

### 4. SysTick Calibration Value Register (SYST_CALIB)

**Address:** `0xE000E01C`

```c
typedef struct
{
    uint32_t TENMS    : 24;    // Bits 0-23: Calibration value (10ms reference)
    uint32_t          : 6;     // Bits 24-29: Reserved
    uint32_t SKEW     : 1;     // Bit 30: Skew flag
    uint32_t NOREF    : 1;     // Bit 31: No reference clock available
} SYST_CALIB_t;
```

**Usage:** Read-only register containing a reference value for calibration. TENMS field contains the number of clock cycles for 10 milliseconds.

## SysTick Configuration

### Basic Configuration Steps

1. **Set the reload value** (determines period)
2. **Clear the current value** (optional, but recommended)
3. **Select clock source**
4. **Enable SysTick interrupt** (if needed)
5. **Enable the counter**

### Detailed Example Configuration

```c
#include <stdint.h>

// Base address of SysTick registers
#define SYST_BASE 0xE000E000

// SysTick Registers
#define SYST_CSR   (*(volatile uint32_t *)(SYST_BASE + 0x10))
#define SYST_RVR   (*(volatile uint32_t *)(SYST_BASE + 0x14))
#define SYST_CVR   (*(volatile uint32_t *)(SYST_BASE + 0x18))
#define SYST_CALIB (*(volatile uint32_t *)(SYST_BASE + 0x1C))

// Bit field definitions
#define SYST_CSR_ENABLE      (1 << 0)
#define SYST_CSR_TICKINT     (1 << 1)
#define SYST_CSR_CLKSOURCE   (1 << 2)
#define SYST_CSR_COUNTFLAG   (1 << 16)

/**
 * Initialize SysTick timer
 * @param ticks: Number of clock cycles between interrupts
 */
void SysTick_Init(uint32_t ticks)
{
    if (ticks > 0xFFFFFF) {
        return;  // Invalid value
    }

    // Set reload value
    SYST_RVR = (ticks - 1);

    // Clear current value
    SYST_CVR = 0;

    // Configure: Enable interrupt, use processor clock, enable counter
    SYST_CSR = SYST_CSR_CLKSOURCE | SYST_CSR_TICKINT | SYST_CSR_ENABLE;
}

/**
 * SysTick interrupt handler
 */
void SysTick_Handler(void)
{
    // Periodic code here
    
}
```

## Practical Examples

### Example 1: 1ms SysTick Interrupt

Assuming a 72 MHz processor clock:

```c
// For 1ms tick:
// Reload = (1ms × 72MHz) - 1 = 72000 - 1 = 71999

#define SYSTICK_1MS_72MHZ  71999

void Init_SysTick_1ms(void)
{
    SysTick_Init(SYSTICK_1MS_72MHZ + 1);
}

// Global millisecond counter
volatile uint32_t milliseconds = 0;

void SysTick_Handler(void)
{
    milliseconds++;
}

uint32_t Get_Milliseconds(void)
{
    return milliseconds;
}
```

### Example 2: Delay Function Using SysTick

```c
/**
 * Delay for specified milliseconds
 * @param ms: Delay time in milliseconds
 */
void Delay_ms(uint32_t ms)
{
    uint32_t start = milliseconds;
    while ((milliseconds - start) < ms) {
        // Wait
    }
}

int main(void)
{
    Init_SysTick_1ms();
    
    while(1) {
        // Do something
        Delay_ms(500);  // Wait 500ms
        // Do something else
    }
}
```

### Example 3: Runtime Frequency Measurement

```c
/**
 * Measure actual processor frequency using SysTick calibration value
 * @return: Frequency in Hz
 */
uint32_t Measure_CPU_Frequency(void)
{
    uint32_t calibration = SYST_CALIB & 0xFFFFFF;  // Get TENMS value
    
    // TENMS represents clock cycles for 10ms
    // Frequency = (calibration / 10ms) = (calibration × 100) Hz
    return (calibration * 100);
}
```

## Common Use Cases

### 1. **RTOS Task Scheduling**
SysTick provides the clock source for task switching in real-time operating systems like FreeRTOS, RTX, and ChibiOS.

### 2. **Performance Profiling**
Use SysTick counter to measure code execution time and profile performance bottlenecks.

### 3. **Periodic Data Sampling**
Collect sensor data at regular intervals with predictable timing.

### 4. **Audio/Signal Processing**
Generate sample-rate clocks for audio applications.

### 5. **System Watchdog**
Implement watchdog-like functionality by tracking SysTick intervals.

### 6. **PWM Generation**
Create software-based PWM signals with SysTick-based timing.

## Important Considerations

### 1. **24-bit Limitation**
Maximum reload value is 0xFFFFFF (16,777,215). For longer delays, use a prescaler or software counter:

```c
// Example: Using a software prescaler
static uint32_t systick_prescaler = 0;

void SysTick_Handler(void)
{
    systick_prescaler++;
    
    if (systick_prescaler >= 1000) {
        // Execute this every 1000 SysTick interrupts
        systick_prescaler = 0;
        // code here
    }
}
```

### 2. **Clock Source Selection**
- **Internal clock (CLKSOURCE=1):** Faster, more accurate, but uses more power
- **External reference (CLKSOURCE=0):** More power-efficient but slower

### 3. **Interrupt Priority**
SysTick typically uses the lowest interrupt priority to allow other interrupts to preempt it:

```c
// Set SysTick priority (typically lowest)
#define SYSTICK_PRIORITY 15

// Initialization code
// (Method varies by microcontroller)
```

### 4. **Critical Section Protection**
When reading `milliseconds` variable from non-interrupt context, protect against updates:

```c
uint32_t Get_Milliseconds_Safe(void)
{
    uint32_t ms;
    __disable_irq();           // Disable interrupts
    ms = milliseconds;
    __enable_irq();            // Re-enable interrupts
    return ms;
}
```