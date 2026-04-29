---
title: ARM Basics
date: 2026-04-21
categories: [ARM]
tags: [arm]
---

ARM (Advanced RISC Machine)<br>
A processor architecture based on Reduced Instruction Set Computing (RISC).

Key Architectural Features:
- **<u>RISC Architecture:</u>**
    Unlike x86 (CISC), which uses complex instructions that can take multiple clock cycles, **ARM (RISC) uses simpler instructions that typically execute in a single cycle.**

- **<u>Load/Store Architecture:</u>**
    In ARM, the CPU cannot perform arithmetic or logical operations directly on data stored in the main memory (RAM).Data must be explicitly loaded from memory into a register, processed, and then stored back to memory.**This separates memory access from data processing, which simplifies the pipeline and improves performance**.

- **<u>Large General-Purpose Register:</u>**
    **ARM provides a generous set of registers**.<br>
    **AArch64 (64-bit):** Features 31 general-purpose registers (X0 to X30), each 64 bits wide.<br>
    **AArch32 (32-bit):** Features 16 general-purpose registers (R0 to R15).