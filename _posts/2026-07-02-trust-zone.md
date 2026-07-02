---
title: "Understanding ARM Cortex-M TrustZone"
date: 2026-07-02 01:00:00
categories: arm
tags: [arm, TrustZone, cortex-m]
---

ARM TrustZone for Cortex-M is the security foundation for modern MCU. It creates two distinct execution worlds Secure and Non-secure, so safety-critical code can be protected from general application code and from external attacker-controlled software.

TrustZone splits a Cortex-M system into two security domains:

- **Secure state**: trusted firmware, cryptographic keys, secure peripherals, and secure data.
- **Non-secure state**: application code, user code, and most peripheral access.

The key property is that secure code can access non-secure memory, but non-secure code cannot access secure memory unless the secure world explicitly exposes a safe interface.

TrustZone is not just a memory partitioning feature. It is a processor-level security model that changes how the core interprets exceptions, stack frames, and branch targets.

## Secure and Non-secure memory

There are three important memory classes:

- **Secure memory**: accessible only when the processor is in secure state.
- **Non-secure memory**: accessible from both secure and non-secure state.
- **Non-secure Callable (NSC) memory**: a small secure region that non-secure code may call into to request services.

A common system layout is:
- Secure bootloader and firmware in secure flash
- Secure data and keys in secure SRAM
- Main application code in non-secure flash
- Non-secure data in non-secure SRAM

## TrustZone register blocks

Cortex-M TrustZone uses a few register blocks for configuring security attribution and access control.

### 1. SAU registers (Security Attribution Unit)

The SAU is the main programmable block that defines which memory regions are secure, non-secure, or non-secure callable.

Key SAU registers:

- `SAU->TYPE`: number of SAU regions supported.
- `SAU->CTRL`: enable flags for the SAU.
- `SAU->RNR`: selects the region number to configure.
- `SAU->RBAR`: region base address.
- `SAU->RLAR`: region limit and security attributes.

#### SAU_CTRL (0xE000EDD0)

This register enables the SAU and controls default memory permissions.

- `ENABLE` (bit 0): Enables the SAU.
- `ALLNS` (bit 1): Treats all memory as non-secure when set.

If `ALLNS` is set, secure state loses the ability to distinguish secure memory regions and the SAU is effectively bypassed for security partitioning.

#### SAU_RNR (0xE000EDD4)

This register selects the active SAU region number for configuration. Configure one region at a time by writing the region index to `SAU_RNR`.

#### SAU_RBAR (0xE000EDD8)

This register holds the base address of the selected region.

- Bits [31:5] define the region base address.
- Bits [4:0] are implementation-defined or reserved.

The base address is aligned to the region size. For example, a 32 KB region must start on a 32 KB boundary.

#### SAU_RLAR (0xE000EDDC)

This register holds the region limit and enables the region.

- Bits [31:5] define the region limit address.
- `NSC` (bit 1): marks the region as Non-secure Callable.
- `ENABLE` (bit 0): enables the region.

When `NSC` is set, the region is still secure memory, but it may be entered from non-secure state through a branch instruction to an NSC gateway.

### 2. NSACR (Non-secure Access Control Register)

`NSACR` controls whether non-secure code can use certain processor features or access some secure-only state.

Important NSACR bits include:

- `CP10` and `CP11`: control non-secure access to the floating-point unit.
- `SMPU`: controls access to secure memory protection resources.

Non-secure software cannot clear all security restrictions through `NSACR`; it only selects which secure features are visible from the non-secure domain.

### 3. Implementation Defined Attribution Unit (IDAU)

Some Cortex-M devices include an IDAU, which is a fixed hardware block for vendor-defined secure and non-secure attribution. Unlike the SAU, the IDAU is not always programmable from software.

The IDAU is often used to define secure partitions for internal memory and peripherals that are locked down by the silicon vendor.

### 4. Secure fault and exception registers

TrustZone adds a secure fault mechanism for security violations.

When a non-secure access violates a secure attribute, the processor can raise a **SecureFault** exception.

Secure fault handling is often visible through the `SCB` system control block, including:

- SecureFault handler entry
- Secure Fault Status registers
- Secure Fault Address register (SFAR)

These registers help diagnose attempts to access secure memory from non-secure state or execute an invalid NSC transition.

## SAU region configuration example

Here is a typical example of configuring SAU regions using CMSIS-style register access.

```c
#include <stdint.h>
#include "core_cm33.h"

#define SAU_RLAR_REGION_Msk  (0xFFFFFFE0UL)
#define SAU_RLAR_NSC_Msk     (1UL << 1)
#define SAU_RLAR_ENABLE_Msk  (1UL << 0)

void sau_configure_region(uint32_t region, uint32_t base, uint32_t limit, bool nsc)
{
    SAU->RNR  = region;
    SAU->RBAR = base & 0xFFFFFFE0UL;
    SAU->RLAR = (limit & 0xFFFFFFE0UL)
               | (nsc ? SAU_RLAR_NSC_Msk : 0U)
               | SAU_RLAR_ENABLE_Msk;
}

void trustzone_init(void)
{
    /* Example region 0: secure flash region */
    sau_configure_region(0, 0x08000000U, 0x0800FFFFU, false);

    /* Example region 1: secure SRAM region */
    sau_configure_region(1, 0x20000000U, 0x20000FFFU, false);

    /* Example region 2: Non-secure callable gateway in secure flash */
    sau_configure_region(2, 0x08010000U, 0x08010FFFU, true);

    /* Enable SAU, keep secure default memory mapping active */
    SAU->CTRL = 1U;
    __DSB();
    __ISB();
}
```

### How region addresses work

The SAU region base and limit values are aligned to 32-byte boundaries, and the region is inclusive of both the base and limit addresses.

A correctly aligned region is required. Otherwise, the SAU may treat the region boundary incorrectly and expose more memory than intended.

## Transition Phase

When non-secure code branches to a secure callable region, the core performs a state transition:

1. The branch target is in an NSC region.
2. The processor automatically switches to secure state.
3. The Secure Gateway (SG) handles the entry and sets up a secure stack frame.
4. When the secure function returns, control goes back to non-secure state.

This transition is critical because it maintains the boundary between trusted and untrusted code.

## Control flow and branch instructions

Cortex-M TrustZone adds special branch behavior via the non-secure state bit in the target address and the exception return value.

- Branches to non-secure code use a non-secure return state.
- Branches to secure code from non-secure state are only allowed through NSC entries.

This means the processor enforces a strict call/return discipline between secure and non-secure domains.

## Security attribute examples

Some common SAU configurations are:

- Secure Flash: secure, executable, not callable from non-secure.
- Secure SRAM: secure, writable, not accessible from non-secure.
- NSC Flash: secure, executable, callable from non-secure.
- Non-secure flash: non-secure, executable by non-secure code.
- Non-secure SRAM: non-secure, read/write by non-secure code.

## Registers Summary

| Register | Purpose | Notes |
|---|---|---|
| `SAU->TYPE` | Number of SAU regions | Read-only hardware capability info |
| `SAU->CTRL` | SAU enable and default state | `ENABLE`, `ALLNS` bits |
| `SAU->RNR` | Region select | Choose which SAU region to program |
| `SAU->RBAR` | Region base address | Base address aligned to region size |
| `SAU->RLAR` | Region limit and attributes | `NSC` and `ENABLE` bits |
| `NSACR` | Non-secure access control | Controls non-secure access to secure features |
| `IDAU` | Implementation-defined attribution | Fixed or vendor-defined secure partitions |

## TrustZone register conventions

A few rules when programming TrustZone registers:

- Always configure all SAU regions before enabling the SAU.
- Use `__DSB()` and `__ISB()` after changing SAU state.
- Set `ALLNS = 0` to keep secure memory secure.
- Use NSC regions only for carefully reviewed gateway functions.
- Keep secure data and key storage inside secure-only regions.

## Debugging TrustZone

Common issues with TrustZone configuration include:

- Incorrect SAU region alignment.
- Overlapping SAU regions.
- Forgetting to enable `SAU->CTRL` after programming regions.
- Placing non-secure code in secure-only memory.
- Creating an NSC region too large or without a strict entry protocol.

When a secure boundary is violated, the processor usually raises a SecureFault. A SecureFault report can tell following things:

- whether the violation was a read, write, or execute attempt
- which address caused the fault
- whether the fault originated in non-secure state.