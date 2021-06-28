---
layout: post
mathjax: true
comments: true
title:  "Summary of 6 modes of STML MCU"
author: John Z. Li
date:   2020-10-10 19:00:18 +0800
categories: c programming
tags: STM8
---
With the following 6 modes of MCU of STML series, below is a summary.
![6 modes of STML](/assets/image/stml8_modes.png)
Note:

Note1: All interrupts must be masked while exiting from **Low Power Run** mode.
Interrupts must not be used to exit this mode.

Note2: While in **Low Power Wait** Mode, all interrupts must be masked.
They must not be used to exit from this mode.

Note3: When woke up from **Low Power Wait** Mode by an event,
the MCU transits to **Low Power Run Mode**.

Note4: In **Low Power Wait** Mode, some interrupts are served if it has been previously enabled.
After processing the interrupts, the processor goes back to Low Power Wait Mode.

Note5: Before executing **HALT**, any pending peripheral interrupt must be
cleared by clearing certain bit in corresponding registers.
Otherwise, **HALT** will fail to be executed.
**HALT** can also be delayed because some flags of certain register are set.

Note6: It is not safe to execute *HALT* from the Low Power Run Mode.

Notation:

WFE: Wait for Event

WFI: Wait for Interrupt

MVR: Main Voltage Regulator mode

LPVR: Low-Power Voltage Regulator mode

RTC: Real-Time Clock

HSE: High Speed External crystal

HSI: High Speed Internal oscillator

LSE: Low Speed External crystal

LSI: Low Speed Internal oscillator

How can *Reset* occur:

1. External reset via the NRST pin

2. POR (Power-On Reset)

3. PDR (Power-Down Reset)

4. IWDG (Independent Watchdog) reset

5. WWDG (Window Watchdog) reset

6. ILLOP (Illegal Opcode) reset

7. SWIM reset
