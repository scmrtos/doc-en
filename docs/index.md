# scmRTOS

The abbreviation **scmRTOS** stands for **Single-Chip Microcontroller Real Time Operating System**.

As the name implies, scmRTOS is designed for single-chip microcontrollers (MCUs), although nothing prevents it from being used with processors like Blackfin or Cortex-A.

### Why?

One of the primary goals behind creating this RTOS was the desire to obtain the simplest, most minimalist, fast, and efficient solution for implementing preemptive multitasking when using single-chip MCUs. Their resources are limited and typically cannot be expanded. Although technological advancements since the emergence of scmRTOS have significantly reduced the demands on RTOS efficiency, simplicity, speed, and small size remain desirable factors in many cases.

The second main reason for scmRTOS's development is its use and implementation in the C++ programming language. C++ has become quite complex today, but scmRTOS employs only the most basic concepts and constructs of the language — essentially what was included in the C++98 standard: classes, templates, inheritance, function overloading.

C++ in the embedded field is often seen as a deterrent — there's a certain mythology surrounding this language (high overhead, poor controllability, etc.). However, in practice, using C++ can simplify, rather than complicate, software development and maintenance (though the opposite effect is possible with the wrong choice of tools for the task).

scmRTOS focuses on ease of use, greatly aided by the concept of a class, which encapsulates properties and provides the user only with an interface. This approach reduces the risk of incorrect usage of operating system objects.

!!! tip "TIP"
    The history of **scmRTOS** and some "philosophical" remarks on the topic of real-time operating systems are provided in the document [scmRTOS.ru.pdf](https://github.com/scmrtos/scmrtos-doc/blob/master/pdf/scmRTOS.ru.pdf).

### Supported Platforms

Currently, scmRTOS supports the following platforms (processor/toolchain):

-   MSP430/IAR Systems;
-   MSP430/GCC;
-   AVR/IAR Systems;
-   AVR/GCC;
-   Cortex-M/GCC;
-   Cortex-M/IAR Systems;
-   Cortex-A/GCC;
-   ARM7/GCC;
-   Blackfin/VisualDSP++;
-   Blackfin/GCC;
-   Blackfin/CCES;
-   STM8/IAR Systems.