# Ports

## General Notes

Due to significant differences in both hardware architectures and the development tools targeting them, adaptation of the OS code[^1] to specific platforms is required. The result of this effort is the platform-specific portion, which, together with the common core, constitutes the complete port for a given platform. The process of preparing the platform-specific portion is referred to as porting.

[^1]: This applies not only to the OS but also to other cross-platform software.

This chapter primarily examines the platform-specific components, their contents and characteristics, and provides brief instructions for porting the OS—i.e., the steps required to create a new port.

The platform-specific code for each target platform is contained in a separate directory and minimally includes three files:

* `os_target.h` – platform-specific declarations and macros.
* `os_target_asm.ext`[^2] – low-level code, including context switch functions and OS startup routines.
* `os_target.cpp` – definitions of the process stack frame initialization function and the interrupt handler for the timer used as the system timer.

[^2]: The file extension for assembly source code specific to the target processor.

Configuration of the OS code for a target platform is achieved through:

* definition of special preprocessor macros;
* conditional compilation directives;
* definition of user-defined types whose implementation depends on the target platform;
* type aliases for certain types;
* definition of functions whose implementation is delegated to the port level.

A critical and delicate part of the port code involves the implementation of assembly-language subroutines responsible for system startup, saving the context of the interrupted process, switching stack pointers, and restoring the context of the process gaining control—including the software interrupt handler that performs context switching. Implementing this code requires the port developer to have in-depth knowledge of the target hardware architecture at a low level, as well as proficiency in using the toolchain (compiler, assembler, linker) for mixed-language[^3] projects.

[^3]: That is, projects containing source files in different programming languages—in this case, C++ and the assembly language of the target hardware platform.

The porting process primarily consists of identifying the required porting objects and implementing the platform-specific code.

## Porting Objects

### Macros

A number of platform-specific macros must be defined. If a macro is not required for a particular port, it should be defined as empty. The list of macros and their descriptions is provided below.

----
```cpp
INLINE
```

Specifies the inlining behavior for functions. Typically consists of a platform-specific unconditional inlining directive combined with the `inline` keyword.

----
OS_PROCESS

Qualifies the executable function of a process. Contains a platform-specific attribute that informs the compiler that the function does not return, allowing preserved[^4] registers to be used without saving them. This reduces code size and stack usage.

[^4]: Registers whose values must be saved before use and restored afterward to prevent corruption of the calling function's context when invoking another function.

----
```cpp
OS_INTERRUPT
```

Contains a platform-specific extension used to qualify interrupt handlers on the target platform.

----
```cpp
DUMMY_INSTR()
```

A macro defining a no-operation instruction for the target processor (typically `NOP`). Used in the context-switch waiting loop of the scheduler (in the software-interrupt-based context switch variant).

----
```cpp
INLINE_PROCESS_CTOR
```

Controls inlining of process constructors. If inlining is desired, this macro should be defined as `INLINE`; otherwise, it should be left empty.

----
```cpp
SYS_TIMER_CRIT_SECT()
```

Used in the system timer interrupt handler to determine whether a critical section is required. This is relevant when the target processor has a prioritized multi-level interrupt controller, where the system timer handler could be unpredictably interrupted by a higher-priority handler accessing the same OS resources.

----
```cpp
CONTEXT_SWITCH_HOOK_CRIT_SECT
```

Determines whether the context switch hook executes within a critical section. It is essential that the hook executes atomically with respect to kernel variable manipulation (particularly `SchedProcPriority`), meaning the scheduler must not be invoked during hook execution. Scheduler invocation can occur from interrupts if the processor has a hardware prioritized interrupt controller and the software context-switch interrupt has lower priority than others. In such cases, the hook code must run in a critical section, and the macro should be defined as `TCritSect cs`. This is a critical consideration&nbsp;—&nbsp;failure to address it properly can lead to elusive runtime errors, requiring careful attention during porting.

----
```cpp
SEPARATE_RETURN_STACK
```

For platforms with a separate return stack, this macro should be defined as 1. For all other platforms, it should be 0.

----
### Types

```cpp
stack_item_t
```

Type alias for the built-in type representing a stack element on the target processor.

----
```cpp
status_reg_t
```

Type alias for the built-in type matching the bit width of the target processor's status register.

----
```cpp
TCritSect
```

Wrapper class for implementing critical sections.

----
```cpp
TPrioMaskTable
```

Class containing a priority mask (tag) table. Used to improve system efficiency. May be omitted on platforms with hardware support for priority tag computation (e.g., a hardware shifter).

----
```cpp
TISRW
```

Wrapper class for interrupt handlers that utilize OS services.

### Functions

```cpp
get_prio_tag()
```

Converts a priority number to its corresponding tag. Functionally equivalent to shifting a 1 left by the priority value.

----
```cpp
highest_priority()
```

Returns the priority number corresponding to the highest-priority process tag in the process map passed as an argument.

----
```cpp
disable_context_switch()
```

Disables context switching. Currently implemented by disabling interrupts.

----
```cpp
enable_context_switch()
```

Enables context switching. Currently implemented by enabling interrupts.

----
```cpp
os_start()
```

Starts the operating system. Implemented in assembly. Receives a pointer to the stack of the highest-priority process and transfers control by restoring its context.

----
```cpp
os_context_switcher()
```

Assembly function that performs direct context switching between processes.

----
```cpp
context_switcher_isr()
```

Interrupt handler for context switching. Implemented in assembly. Saves the context of the interrupted process, switches stack pointers via a call to `context_switch_hook()`[^5], and restores the context of the activated process.

[^5]: Through the wrapper function `os_context_switch_hook()`, which has `"extern C"` linkage.

----
```cpp
TBaseProcess::init_stack_frame()
```

Initializes the stack frame of a process, arranging memory cells such that the stack appears as if the process was interrupted and its context saved. Used by the process constructor and during process restart.

----
```cpp
system_timer_isr()
```

System timer interrupt handler. Calls `TKernel::system_timer()`.

## Porting Guidelines

Porting typically requires defining all the macros, types, and functions listed above for the target platform.

The most delicate and critical tasks involve implementing the assembly code and the stack frame initialization function. Key aspects requiring particular attention include:

* determining the calling conventions used by the compiler to identify which registers (or stack areas) are used for passing arguments of various types;
* understanding how the processor handles saving of return addresses and status registers upon interrupt occurrence—this is essential for correct stack frame formation on the target platform, which in turn is necessary for implementing context switch functions/handlers and stack frame initialization;
* verifying the name mangling scheme for exported/imported symbols in assembly. In the simplest case, C names (and `"extern C"` names in C++) are visible unchanged in assembly, but on some platforms[^7] prefixes and/or suffixes may be added, requiring assembly functions to be named accordingly for correct linker resolution.

[^7]: Notably on Blackfin.

[^6]: C++ names undergo special mangling to support function overloading and type-safe linking, making direct assembly access difficult. Therefore, functions defined in C++-compiled files that need to be accessed from assembly must be declared with `"extern C"` linkage.

All assembly code should be placed in the file `os_target_asm.ext` mentioned earlier. Macro and type definitions, along with inline functions, belong in `os_target.h`. The file `os_target.cpp` should declare type objects if needed (e.g., `OS::TPrioMaskTable OS::PrioMaskTable`) and define `TBaseProcess::init_stack_frame()` and the system timer interrupt handler `system_timer_isr()`.

The above provides only general information related to OS ports. Porting involves numerous specific nuances whose detailed description is beyond the scope of this document.

!!! tip "**TIP**"
    When creating a new port, it is advisable to use an existing port as a template or reference—this significantly simplifies the process. The choice of reference port depends on the architectural and toolchain similarity between the existing port and the target platform.

## Integration into a Working Project

To enhance flexibility and efficiency, certain platform-specific code that depends on project-specific details and the particular microcontroller used is delegated to the project level. This typically includes selection of the hardware timer used as the system timer and, where applicable, the context-switch interrupt when the processor lacks a dedicated software interrupt.

For port configuration, the project must include the following files:

* `scmRTOS_config.h`;
* `scmRTOS_target_cfg.h`;

The file `scmRTOS_config.h` contains most configuration macros defining parameters such as the number of processes, context switch method, enabling of system time functions, user hook support, priority value ordering, and similar settings.

The file `scmRTOS_target_cfg.h` contains code for managing target processor resources selected for system functions—primarily the system timer and context-switch interrupt.

The contents of both configuration files are described in detail in documents specific to individual ports.