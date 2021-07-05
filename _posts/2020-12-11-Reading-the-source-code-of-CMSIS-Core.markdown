---
layout: post
mathjax: true
comments: true
title:  "Reading the source code of CMSIS-Core(M)"
author: John Z. Li
date:   2020-12-11 19:00:18 +0800
categories: c programming
tags: STM32, CMSIS
---
**The Cortex Microcontroller Software Interface Standard (CMSIS)**
is a specification for a hardware abstraction layer for
ARM Cortex-M based MCUs,
which is also used to refer to any implementation
that is compliant with the specification.
The purpose of CMSIS is to make porting software between
different ARM Cortex-M based MCUs easy.
Note that ARM now uses the term CMSIS to refer to many things,
even including an implementation of RTOS.
we use the term in this post in its narrow sense, that is, the Core(M) part.

The overall structure of an application is like below:
![Conceptual Structure](/assets/image/stm32.png)

CMSIS can be divided into two basic function components

1. Core Peripheral Access Layer (CPAL): An abstraction layer for common components and functionality
that are available for every Cortex system (core registers, NVIC, debug, etc.).
Toolchain specific inline functions to access registers of the MCU. This part is implemented by ARM.
2. Device Peripheral Access Layer (DPAL): Like CPAL,
but for peripherals that may act differently with MCUs from different vendors.
This Module is provided by the vendor of the MCU.
Middleware for those peripherals are supposed to build upon this layer.

**File structure** (assume that we are using a STM32F779 MCU)

1. Top level device description header “stm32f7xx.h”,
which uses macro definitions to select the concrete device header,
that is “stm32f779xx.h”,
which in turn includes “core_cm7.h”, “system_stm32f7xx.h” and “stdint.h”.
(*The top-level header also includes "stm32f7xx_hal.h"
if macro “USE_HAL_DRIVER” is defined,
which is not part of CMSIS*.)
Vendor specific CMSIS version information for STMF7 MCUs is also provided
by macros in this header,
as well as a set of bit manipulation macros and a set of
bool-valued macro pairs like `SET` and `RESET`, `ENABLE` and `DISABLE`,
`ERROR` and `SUCCESS`. The overall structure of the top-level header is like below:
![stm32-header](/assets/image/stm32_header.png)

2. The concrete device header “stm32f779xx.h”,
that implements DPAL, by the vendor of the MCU,
that is ST in this case. This header contains the following:
  - MCU functionality presence specification macros,
    like whether there is an MPU,
    how many bits the NVIC uses to set priority levels,
    whether there is an FPU, etc.
  - It also defines a set of auxiliary types for MCU and core peripheral controls,
    named by the convention `<Functionality>_Type` for enums and `<Functionality>_TypeDef`
    for structs using typedefs,
    for example, `IRQn_Type` is an enum type that
    gives each interrupt a human readable name.
    Registers of core peripherals are also given names by typedef-ing
    corresponding structs that contains register names as identifiers,
    these structs having their fields so arranged that they align to actual
    memory mapped registers addresses in memory given correct base addresses.
    Thus, each core peripheral is given a unique name that is a pointer to
    the base address of its registers.
    Bit level access to registers are facilitated by defining macros expanding to
    integers that can be used to unmasking the corresponding bits.
    Macros with the name of the form `IS_<functionality>_INSTANCE(__INSTANCE__)`
    is provided to check if such a pointer actually points to belongs a certain peripheral as its name suggests.

3. The header “core_cm7.h” defines interface for the CPAL (provided by ARM)
This header includes the following:
  - “cmsis_compiler.h”: abstracts compiler intrinsic directives,
    which may in turn includes a header named as “cmsis_<compiler>.h”.
    Since we are using the GCC toolchain, it includes “cmsis_gcc.h”.
  - “cmsis_version.h”: defines CMSIS version for the APAL.
  - “mpu_armv7.h” if the MCU has an MPU module, which is specified in
    “stm32f779xx.h” as mentioned above. Its header provides APIs to access the MPU module.

This header provides the following:

a. Typedefs to access core registers
    	(Application Program Status Register,
	   Interrupt Program Status Register,
	   Special-Purpose Program Status Registers,
	   Control Registers, NVIC Registers,
	   System Control Block Registers, etc, with “<funtionality>_Type” names.)
b. A set of inline functions that manipulate interrupts and exceptions all starting with “__IVNC”.

```c
        void __NVIC_SetPriorityGrouping(uint32_t(0..7)); // set the priority grouping field SCB->AIRCR
        uint32_t __NVIC_GetPriorityGrouping(void);
        void __NVIC_EnableIRQ(IRQn_Type(enum));
        uint32_t __NVIC_GetEnableIRQ(IRQn_Type(enum)); // check if corresponding interrupt enabled
        void __NVIC_DisableIRQ(IRQn_Type(enum));
        uint32_t __NVIC_GetPendingIRQ(IRQn_Type(enum)); //check whether interrupt status is pending
        void __NVIC_SetPendingIRQ(IRQn_Type(enum)); // set pending bit of a device
        void __NVIC_ClearPendingIRQ(IRQn_Type(enum));
        uint32_t __NVIC_GetActive(IRQn_Type(enum)); // check if interrupt status is active
        void __NVIC_SetPriority(IRQn_Type(enum), uint32_t(0..15));
        uint32_t __NVIC_GetPriority(IRQn_Type(enum));
        uint32_t NVIC_EncodePriority (uint32_t, uint32_t, uint32_t);
        void NVIC_DecodePriority (uint32_t, uint32_t, uint32_t* const, uint32_t* const);
        void __NVIC_SetVector(IRQn_Type(enum), uint32_t(fun_ptr));
        uint32_t(fun_ptr) __NVIC_GetVector(IRQn_Type(enum));
        void __NVIC_SystemReset(void);
```
c. A function that get FPU type of the MCU (if it has one)

d. Functions that set instruction and data cache starting with “SCB_”

e. A function that set system timer and its interrupt, after calling, the timer starts to tick: uint32_t SysTick_Config(uint32_t ticks);

f. Functions that access the debugger interface.

g. The header “system_stm32f7xx.h” is intended to manage the system-wide resources,
that is, device initialization and system core clock update. It declares 3 global variables

```c
        uint32_t SystemCoreClock; //system clock frequency
        const uint8_t  AHBPrescTable[16];
        const uint8_t  APBPrescTable[8];
```
which are defined in the corresponding .c file.
There are also two functions declared:
```c
    extern void SystemInit(void);
    extern void SystemCoreClockUpdate(void); //update core clock frequency.
```
The names are self explanatory.

**Programming Concerns**

1. Coding Style

- CMSIS code is [MISRA-C 2004](https://en.wikipedia.org/wiki/MISRA_C#MISRA_C:2004) compliant.
This implies any extension of CMSIS should be compliant too.

- Name convention; mix of Pascal Casing and underscores, like “SysTick_Config()” and “ITM_RxBuffer”, and macros being of all capital letters.

- External interrupt handlers end with the suffix “_IRQHandler”, and exception handlers end with the suffix “_Handler”.

- Use Doxygen to comment code. Note that C++ style inline comments starting with “//” is forbidden by MISRA-C 2004.

2. Functions in CPAL are reentrant, meaning they can be used in different interrupt handlers safely

3. There are also something that is like [SAL annotations of MS C/C++](https://docs.microsoft.com/en-us/cpp/code-quality/using-sal-annotations-to-reduce-c-cpp-code-defects?view=msvc-160)  but less comprehensive.
The idea is that tools can automatically extract information for sanitization.
The following annotations are introduced by defining macros:

>    __I : read only permission

>    __O : write only permission
>
>    __IO : read/write permission
>
>    __IM : read only of a struct member
>
>    __OM : write only of a struct member
>
>    __IOM : read/write of a struct member

4. A Serial Wire Viewer (SWV) can be used to receive
ITM (Instumentation Trace Macrocell) data through the SWO pin,
which is built-in debug facilities for ARM Cotex-M controllers.
By implementing the weak function that is needed for the corresponding
syscall which is further needed by a libc implementation like newlib,
the standard output can be redirected to the channel 0 of the SWV,
for example, with STMCubeIDE, it can be done by providing an implementation of `int __io_putchar(int ch)` by
```c
    int __io_putchar(int ch){
      return ITM_SendChar(c);
    }
```
