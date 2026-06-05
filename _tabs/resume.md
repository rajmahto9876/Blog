---
title: Resume
icon: fas fa-file
order: 2
---

## My Resume

[Resume]({{ '/assets/Resume.pdf' | relative_url }})

# Professional Summary

Embedded Software Engineer with more than 5 years of experience in Embedded C development, micro-controller firmware, peripheral driver development, and board bring-up.
 
Currently expanding expertise in Embedded Linux, Linux Kernel Internals, Device Drivers, Device Tree, QEMU, and kernel debugging.
 
Hands-on experience with:

- Embedded C
- ARM Cortex-M
- UART, SPI, I2C, Timer, PWM
- RTOS concepts (Free RTOS)
- Linux Kernel Modules
- Character Device Drivers
- Device Tree
- QEMU
- Git
- JTAG Debugging, GDB, Valgrind and KGDB.
 
# Education

**National Institute of Technology, Warangal**\
PG Diploma (M.Tech) in Control and Automation &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  CGPA: 8.43

**Maharaja Agrasen Institute of Technology, Delhi**\
B.Tech in Electrical and Electronics Engineering  &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; CGPA:7.99

# Technical Skills

**Programming Languages:** Embedded C, C++, Python\
**Embedded Systems:** Bare-metal, Startup code, SoC Bring-up, Board
Bring-Up\
**MCU / SoCs:** ARM Cortex-M, ARM Cortex-A, NXP S32K5xx, ESP32, STM32,
Atmel, Basic RiscV.\
**Protocols & Networking:** UART, SPI, I2C, DMA, MQTT, UDP, HTTPS, Wi-Fi
(ESP32)\
**Operating Systems:** FreeRTOS, Linux\
**Embedded Linux:** Linux Kernel Modules, Character Device Drivers,
Memory-Mapped I/O, Cross Compilation, Kernel Debugging, User-Kernel
Communication\
**Build & Toolchains:** CMake, LLVM, Makefiles, Buildroot (basic
exposure), Keil IDE, MPLABX\
**Debug & Validation Tools:** Lauterbach Trace32, Veloce Emulation, GDB,
Valgrind\
**Version Control:** Git, Bitbucket, code reviews, collaborative
development workflows\
**Hardware tools:** Oscilloscope, Multimeter, Logic Analyzer, ST-Link,
ULINK\
**Simulation tools:** Matlab, Simulink\
**Code Quality & Compliance:** MISRA-C coding guidelines, Static code
analysis.

# Linux Skills
- **Linux Device Driver:** GPIO, SPI, I2c, Character Device Drivers, Linux Kernel Modules
- **Linux Internals:** OS-Memory, Threads, Processes, pipes, Shared Memory.
- **Debugging Tools:** GDB, Valgrind.

# Professional Experience

**VVDN Technologies (Onsite: NXP Semiconductors)** &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;May 2024 -- July
2025\
*Senior Software Engineer*

- Designed, developed, tested, and debugged embedded firmware for
  ARM-based automotive SoCs including peripheral validation, interrupt
  verification, timer subsystem testing, and low-level register
  debugging.
- Performed pre-silicon validation on the Veloce emulation platform,
  validated 8+ IP blocks some of them are interrupt controllers, timers,
  clock modules, and reset logic. Utilized Lauterbach Trace32 (T32) for
  register-level debugging, instruction tracing.
- Identified and resolved multiple logic and integration issues prior to
  silicon tape-out, improving hardware--software stability. Automated
  testing reduced regression time by 20%

**IoTfy Solutions Pvt. Ltd., Delhi** &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;May 2021 -- May 2024\
*Tech Lead -- Embedded Systems*

- Developed Linux kernel modules and user-space applications for
  hardware interaction using memory-mapped I/O and device interfaces.
- Mentored and guided a team of embedded engineers, enforcing coding
  standards, design reviews, and best practices.
- Designed **FreeRTOS-based task** architecture using queues,
  semaphores, mutexes, and software timers.

**Copper Connection Pvt. Ltd., Delhi** &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;April 2020 -- May 2021\
*Embedded Engineer*

- Worked on embedded firmware development and feature implementation for
  PBX systems.
- Supported hardware prototyping, debugging, and system validation
  activities.

# Key Projects
**Linux Device Driver on BeagleBone Black** &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;Dec 2025 -- April 2026
- Developed Linux Kernel Modules on BeagleBone Black to understand Linux driver architecture, kernel APIs, and ARM-based embedded systems.
- Implemented MMIO-based GPIO drivers using ioremap(), readl(), and writel() for direct peripheral register access.
- Developed interrupt-driven drivers using request_irq() and interrupt handlers, enabling asynchronous event processing from GPIO peripherals.
- Created Device Tree overlays for custom hardware integration and peripheral configuration.
- Developed SPI-based sensor drivers, handling device initialization, register configuration, and sensor data acquisition through the Linux SPI subsystem.
- Implemented I2C-based OLED display drivers using the Linux I2C framework and verified communication through kernel-space applications.
- Worked with platform drivers and Device Tree matching using compatible strings and probe/remove callbacks.
- Performed kernel debugging using printk(), dmesg, GDB, and KGDB to analyze driver initialization and runtime issues.
- Built and booted custom Linux kernels in QEMU to understand boot flow, kernel configuration, and driver integration.
- Practiced board bring-up activities including Device Tree configuration, GPIO validation, interrupt verification, and peripheral testing.


**Hardware Security Engine (HSE) -- Pre-Silicon Project** &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;May 2024 --
June 2025

- Developed and maintained **CMake build scripts** for compiling
  bare-metal firmware in a multi-module environment. Worked on low-level
  firmware development and debugging for RISC-V based embedded
  platforms.
- Worked extensively on **linker scripts and startup files** for system
  initialization and memory mapping. Enabled and validated critical
  **bare-metal security and system IPs** including Interrupt, Timer,
  Clock Subsystems, MC_RGM, and MC_ME.
- Implemented and executed **unit and system-level test cases** to
  verify firmware correctness and IP behavior. Performed **system
  testing in a pre-silicon environment** using the Veloce emulation
  platform.

**IoT Switches, Plugs, Wi-Fi Mesh Network Implementation (ESP32 &
Chimney Systems** &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;May 2023 -- May 2024

- Designed and implemented a Wi-Fi Mesh network supporting 40+ devices
  under a single IP.
- Designed and developed an **IoT-enabled smart devices** with mobile
  app, HTTP, web server, UDP, MQTT, and cloud integration.
- Strong experience in bare-metal firmware, RTOS-based embedded systems,
  and low-level hardware interfacing.
- Enabled real-time monitoring and remote control through
  cloud-connected embedded firmware.

**BLDC Fan Development** &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;May 2021 -- May 2024

- Optimized BLDC fan tuning and wattage control algorithms to achieve a
  **5-star efficiency rating**.
- Assisted in hardware design activities including **PCB layout** and
  **SMPS integration**. Implemented **low-voltage and high-voltage
  cut-off protection circuits** to enhance system safety and
  reliability.
- Developed **multi-mode control functionality**, supporting IR remote,
  RF remote, and cloud-based operation.

# Languages
- English -- Professional Working Proficiency
- German -- Basic A1 Proficiency