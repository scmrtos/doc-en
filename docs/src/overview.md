# Operating System Overview

## General Information

**scmRTOS** is a real-time operating system featuring priority-based preemptive multitasking. The OS supports up to 32 processes (including the system **IdleProc** process, i.e., up to 31 user processes), each with a unique priority. All processes are static, meaning their number is defined at the project build stage and they cannot be added or removed at runtime.

The decision to forgo dynamic process creation is driven by resource conservation considerations, as resources in single-chip microcontrollers are highly limited. Dynamic process deletion is also not implemented, as it offers little benefit—the program memory used by the process is not freed, and RAM for subsequent use would require allocation/deallocation via a memory manager, which is a complex component that consumes significant resources and is generally not used in single-chip microcontroller projects[^1].

In the current version, process priorities are also static: each process is assigned a priority at the project build stage, and the priority cannot be changed during program execution. This approach is motivated by the goal of making the system as lightweight as possible in terms of resource requirements while maintaining high responsiveness. Changing priorities during system operation is a non-trivial mechanism that, for correct operation, requires analyzing the state of the entire system (kernel, services) followed by modifications to kernel components and other OS parts (semaphores, event flags, etc.). This inevitably leads to prolonged periods with interrupts disabled, significantly degrading the system's dynamic characteristics.

[^1]: This refers to the standard memory manager typically provided with development tools. There are situations where program operation requires storing data between function calls (i.e., automatic storage on the stack or in CPU registers is unsuitable), and the amount of such data is unknown at compile time—their creation and lifetime are determined by events occurring at runtime. The best approach for storing such data is in free memory—the "heap." These operations are usually handled by a memory manager. Thus, some applications cannot do without it, but given the resource consumption of standard memory managers, their use is often unacceptable. In such cases, a specialized memory manager tailored to the application's needs is frequently employed. Considering the above, creating a universal memory manager equally suitable for diverse projects is impractical, which explains the absence of a memory manager in **scmRTOS**.

## OS Structure

The system consists of three main components: the kernel, processes, and inter-process communication services.

### Kernel

The kernel handles:

* Process organization functions;
* Scheduling at both process and interrupt levels;
* Support for inter-process communication;
* System time support (system timer);
* Extension support.

For more details on the kernel's structure, composition, functions, and mechanisms, see the ["Kernel" section](kernel.md).

### Processes

Processes enable the creation of independent (asynchronous relative to others) threads of execution in the program. Each process provides a function that must contain an infinite loop serving as the process's main loop—see "Listing 1. Process Execution Function" for an example.

<a name="process-exec"></a>
```cpp
template<> void slon_proc::exec()
{
    ... // Declarations
    ... // Init process’s data
    for(;;)
    {
        ... // process’s main loop
    }
}
```

/// Caption
Listing 1. Process Execution Function
///

Upon system startup, control is transferred to the process function, where declarations of used data (line 3) and initialization code (line 4) can be placed at the beginning, followed by the process's main loop (lines 5–8). User code must be written to prevent exiting the process function. For example, once entering the main loop, do not leave it (the primary approach), or if exiting the main loop, enter another loop (even an empty one) or an infinite "sleep" by calling the `sleep()` function[^2] without parameters (or with parameter "0")—see [The `sleep()` Function](processes.md#process-sleep) for details. The process code must not contain `return` statements.

[^2]: In this case, no other process should "wake" this sleeping process before exit, as it would lead to undefined behavior and likely cause the system to crash. The only safe action applicable to a process in this state is to terminate it (with the option to restart from the beginning); see [Process Restart](processes.md#process-restart).

### Inter-Process Communication

Since processes execute in parallel and asynchronously relative to each other, simply using global data for exchange is incorrect and dangerous: while one process accesses an object (which could be a built-in type variable, array, structure, class object, etc.), it may be preempted by a higher-priority process that also accesses the same object. Due to the non-atomic nature of access operations (read/write), the second process could corrupt the first process's actions or simply read incorrect data.

To prevent such issues, special measures are required: access within critical sections (where context switching is disabled) or using dedicated inter-process communication services. In **scmRTOS**, these include:

* Event flags (`OS::TEventFlag`);
* Mutual exclusion semaphores (`OS::TMutex`);
* Channels for data transfer as queues of bytes or arbitrary-type objects (`OS::channel`);
* Messages (`OS::message`).

The developer must decide which service (or combination) to use in each case, based on task requirements, available resources, and personal preferences.

Starting with **scmRTOS v4**, inter-process communication services are built on a common specialized class `TService`, which provides all necessary base functionality for implementing service classes/templates. This class's interface is documented and intended for users to extend the set of services by designing and implementing custom inter-process communication mechanisms best suited to specific project needs.

## Software Model

### Composition and Organization

The **scmRTOS** source code in any project consists of three parts: common (core), platform-dependent (target), and project-dependent (project).

The common part contains declarations and definitions for kernel functions, processes, system services, plus a small support library with useful code, some of which is directly used by the OS.

The platform-dependent part includes declarations and definitions specific to the target platform, compiler language extensions, etc. This encompasses assembly code for context switching and system startup, the stack frame initialization function, the critical section wrapper class definition, the interrupt handler for the hardware timer used as the system timer, and other platform-specific behavior.

The project-dependent part consists of three header files with configuration macros, extension inclusions, and optional code for fine-tuning the OS to the specific project—such as type aliases for timeout variable bit widths, selection of the context switch interrupt source, and other means for optimal system operation.

Recommended file placement: common part in a separate `core` directory, platform-dependent part in a `<target>` directory (where `target` is the name of the target port), and project-dependent part directly in the project source files. This layout facilitates storage, portability, maintenance, and safer updates when upgrading to new versions.

The common part source code is in eight files:

* `scmRTOS.h` – main header file, including the entire system header hierarchy.
* `os_kernel.h` – primary kernel type declarations and definitions.
* `os_kernel.cpp` – kernel object declarations and function definitions.
* `scmRTOS_defs.h` – auxiliary declarations and macros.
* `os_services.h` – service type and template definitions.
* `os_services.cpp` – service function definitions.
* `usrlib.h` – support library type and template definitions.
* `usrlib.cpp` – support library function definitions.

As evident from the list, **scmRTOS** includes a small support library containing code used by OS components[^3]. Since this library is not essentially part of the OS, it will not be discussed further here.

[^3]: In particular, the ring buffer class/template.

The platform-dependent part source code is in three files:

* `os_target.h` – platform-dependent declarations and macros.
* `os_target_asm.ext`[^4] – low-level code for context switching and OS startup functions.
* `os_target.cpp` – process stack frame initialization function definition, system timer interrupt handler, and the idle process root function.

[^4]: Assembly file extension for the target processor.

The project-dependent part consists of three header files:

* `scmRTOS_config.h` – configuration macros and type aliases, particularly for timeout object bit widths.
* `scmRTOS_target_cfg.h` – code for tailoring OS mechanisms to the project; e.g., specifying the interrupt vector for the system timer handler, system timer control macros, context switch interrupt activation function definition, etc.
* `scmRTOS_extensions.h` – extension inclusion control. See [TKernelAgent and Extensions](kernel.md#kernel-agent) for details.

### Internal Structure

Everything related to **scmRTOS**, except a few assembly-implemented functions with `extern "C"` linkage, is placed inside the `OS` namespace—providing a dedicated namespace for OS components.

Within this namespace, the following classes are declared[^5]:

* `TKernel`. Since only one kernel instance exists, there is only one object of this class. Users should not create instances;
* `TBaseProcess`. Implements the base object type for the `process` template, on which all (user or system) processes are built;
* `process`. Template for creating types of any OS process.
* `TISRW`. Wrapper class to simplify and automate interrupt handler code creation. Its constructor handles entry actions, and destructor handles exit actions.
* `TKernelAgent`. Special service class providing access to kernel resources for extending OS capabilities. It forms the basis for `TService` (base for all inter-process communication services) and the profiler class.

[^5]: Nearly all OS classes are declared as friends of each other to ensure access among OS components to each other's internals.

The service classes include:

* `TService`. Base class for all inter-process communication types and templates. Provides common functionality and defines the application programming interface (API) for derived types. Serves as the foundation for extending communication facilities;
* `TEventFlag`. For inter-process interaction via binary semaphore (event flag) signaling;
* `TMutex`. Binary semaphore for mutual exclusion access to shared resources;
* `message`. Template for message objects. Similar to event flags but can carry an arbitrary-type payload (usually a structure);
* `channel`. Template for data channels of arbitrary types. Basis for message queues.

Note that counting semaphores are absent from the list, as no compelling need for them was identified. Resources requiring counting semaphore control—primarily RAM—are in short supply in single-chip microcontrollers. Situations needing quantity tracking are handled using objects based on the `OS::channel` template.

As evident from the list above, counting semaphores are absent. The reason is that, despite every effort, no compelling need for them could be identified. Resources requiring control via counting semaphores—primarily RAM—are in acute shortage in single-chip microcontrollers. Situations where tracking the available quantity is still necessary are handled using objects based on the `OS::channel` template, which already implement the corresponding mechanism in one form or another. If such a service is needed, the user can add it to the base set independently by creating their own implementation as an extension; see [TKernelAgent and Extensions](kernel.md#kernel-agent).

**scmRTOS** provides the user with several functions for control:

* `run()`. Intended for starting the OS. When this function is called, the actual operation of the operating system begins—control is transferred to the processes, whose execution and mutual interaction are determined by the user program. After transferring control to the OS kernel code, the function does not regain it, and therefore no return from the function is provided;
* `lock_system_timer()`. Blocks interrupts from the system timer. Since the selection and handling of the hardware part of the system timer are the responsibility of the project, the user must define the content of this function. The same applies to the paired function `unlock_system_timer()`;
* `unlock_system_timer()`. Unblocks interrupts from the system timer;
* `get_tick_count()`. Returns the number of system timer ticks. The system timer tick counter must be enabled during system configuration;
* `get_proc()`. Returns a pointer to the constant process object by the index passed as an argument to the function. The index is effectively the process priority value.

### Critical Sections

Due to the preemptive nature of process execution, any process can be interrupted at an arbitrary moment. On the other hand, there are cases[^6] where it is necessary to prevent a process from being interrupted during the execution of a specific code fragment. This is achieved by disabling context switching[^7] for the duration of that fragment's execution. In other words, this fragment acts as a non-interruptible section.

[^6]: For example, accessing OS kernel variables or representations of inter-process communication services.

[^7]: In the current version of **scmRTOS**, this is achieved by globally disabling interrupts.

In OS terms, such a section is called a critical section. To simplify the organization of a critical section, a special wrapper class `TCritSect` is used. Its constructor saves the state of the processor resource controlling global interrupt enable/disable and then disables interrupts. The destructor restores this processor resource to the state it was in before the interrupts were disabled.

Thus, if interrupts were already disabled, they remain disabled. If they were enabled, they are re-enabled. The implementation of this class is platform-dependent, so its definition is contained in the corresponding file `os_target.h`.

Using `TCritSect` is straightforward: at the point corresponding to the start of the critical section, simply declare an object of this type, and from the declaration point until the end of the block, interrupts will be disabled[^8].

[^8]: Upon exiting the block, the destructor is automatically called, restoring the state that existed before entering the critical section. This approach eliminates the possibility of "forgetting" to re-enable interrupts when exiting the critical section.

### Type Aliases for Built-in Types

To facilitate working with source code and improve portability, the following type aliases are introduced:

* `TProcessMap` – type for defining a variable that serves as a process map. Its size depends on the number of processes in the system. Each process corresponds to a unique tag—a mask with only one non-zero bit positioned according to the process's priority. The highest-priority process corresponds to the least significant bit (position 0)[^9]. With fewer than 8 user processes, the process map size is 8 bits. With 8 to 15, it is 16 bits; with 16 or more user processes, it is 32 bits.
* `stack_item_t` – type for a stack element. Depends on the target architecture. For example, on 8-bit **AVR**, this type is defined as `uint8_t`; on 16-bit **MSP430**, as `uint16_t`; and on 32-bit platforms, typically as `uint32_t`.

[^9]: This order is the default. If `scmRTOS_PRIORITY_ORDER` is defined as 1, the bit order in the process map is reversed—the most significant bit corresponds to the highest-priority process, and the least significant bit to the lowest-priority one. Reverse priority order can be useful for processors with hardware support for finding the first non-zero bit in a word, such as the **Blackfin** family.

### Using the OS

As noted earlier, to achieve maximum efficiency, static mechanisms are used wherever possible—i.e., all functionality is determined at compile time.

This primarily concerns processes. Before using each process, its type must be defined[^10], specifying the process type name, its priority, and the size of the RAM area allocated for the [process stack](). For example:

[^10]: Each process is an object of a separate type (class) derived from the common base class `TBaseProcess`.

```cpp
OS::process<OS::pr2, 200> MainProc;
```

This defines a process with priority `pr2` and a stack size of 200 bytes. Such a declaration may seem somewhat verbose due to its length, as referencing the process type requires writing the full expression—for example, when defining the process execution function[^11]:

[^11]: The execution function of a specific process is technically a full specialization of the `OS::process::exec()` template member function, so its definition uses the template specialization syntax `template<>`.

```cpp
template<> void OS::process<OS::pr2, 200>::exec() { ... }
```

because the type is precisely the expression

```cpp
OS::process<OS::pr2, 200>
```
A similar situation arises in other cases where referencing the process type is required. To eliminate this inconvenience, it is recommended to use **type aliases** introduced via `typedef` or `using`. This is the preferred coding style: first define type aliases for processes (preferably in a single header file for easy overview of all processes in the project), and then declare the actual process objects in the source files as needed. With this approach, the earlier example becomes[^12]:

[^12]: It is recommended to declare a prototype of the process execution function specialization before the first instantiation of the template—this allows the compiler to recognize that a full specialization exists for that instance, avoiding attempts to generate the default template implementation. In some cases, this prevents compilation errors.

```cpp
// In a header file
typedef OS::process<OS::pr2, 200> TMainProc;
...
template<> void TMainProc::exec();

// In a source file
TMainProc MainProc;
...
template<> void TMainProc::exec()
{
    ...
}
...
```

There is nothing unusual about this sequence—it is the standard way of defining a type alias and creating an object of that type in C and C++.

!!! warning "**IMPORTANT NOTE**"
    When configuring the system, the number of processes must be explicitly specified. This number must exactly match the number of processes actually defined in the project; otherwise, the system will not function correctly. Note that priorities are specified using a dedicated enumerated type `TPriority`, which defines the allowed priority values[^13].

    Additionally, process priorities must be consecutive with no gaps. For example, if the system has 4 processes, their priorities must be `pr0`, `pr1`, `pr2`, and `pr3`. Duplicate priority values are also not allowed—each process must have a unique priority.

    For instance, with 4 user processes (resulting in 5 processes total, including the system `IdleProc`), the priorities should be `pr0`, `pr1`, `pr2`, `pr3` (`prIDLE` is reserved for `IdleProc`), where `pr0` is the highest-priority process and `pr3` is the lowest-priority user process. The lowest-priority process overall is always `IdleProc`. This process exists permanently in the system and does not need to be declared. It receives control whenever all user processes are inactive.

    The compiler does not check for gaps in priority numbering or for priority uniqueness, as—following the principle of separate compilation—there is no efficient way to automate such configuration integrity checks using language features alone.

    A dedicated tool currently exists to perform comprehensive configuration integrity checking. The utility is called **scmIC** (Integrity Checker) and can detect the vast majority of typical OS configuration errors.

[^13]: This is done to improve type safety—arbitrary integer values cannot be used; only those defined in `TPriority` are permitted. The values in `TPriority` are tied to the process count specified via the configuration macro `scmRTOS_PROCESS_COUNT`. Thus, only a limited, valid set of priorities is available. Priority values take the form `pr0`, `pr1`, etc., where the number indicates the priority level. The system `IdleProc` process has its own dedicated priority designation: `prIDLE`.

As mentioned earlier, defining process types in a header file is convenient, as it makes any process easily visible across different compilation units.

An example of typical process usage is shown in "Listing 2. Defining Process Types in a Header File" and "Listing 3. Declaring Processes in a Source File and Starting the OS".

```cpp
 1 //------------------------------------
 2 //
 3 // Process types definition
 4 //
 5 //
 6 typedef OS::process<OS::pr0, 200> TUARTDrv;
 7 typedef OS::process<OS::pr1, 100> TLCDProc;
 8 typedef OS::process<OS::pr2, 200> TMainProc;
 9 typedef OS::process<OS::pr3, 200> TFPGA_Proc;
10 //-------------------------------------
```

/// Caption
Listing 2. Defining Process Types in a Header File
///

```cpp
 1 //-------------------------------------
 2 //
 3 // Processes declarations
 4 //
 5 //
 6 TUartDrv UartDrv;
 7 TLCDProc LCDProc;
 8 TMainProc MainProc;
 9 TFPGAProc FPGAProc;
10 //-------------------------------------
11
12 //-------------------------------------
13 void main()
14 {
15     ... // system timer and other stuff initialization
16     OS::run();
17 }
18 //-------------------------------------
```

/// Caption
Listing 3. Declaring Processes in a Source File and Starting the OS
///

Each process, as mentioned earlier, has an executable function. When using the scheme described above, this executable function is named `exec` and looks as shown in "Listing 1. Process Execution Function".

Configuration information is specified in a dedicated header file `scmRTOS_config.h`. The list of configuration macros and their meanings[^14] are provided in "Table 1. Configuration Macros".

[^14]: The table shows example values. In each project, values are set individually based on project requirements.

| Name                                      | Value   | Description                                                                 |
|-------------------------------------------|---------|-----------------------------------------------------------------------------|
| `scmRTOS_PROCESS_COUNT`                   | n       | Number of processes in the system                                           |
| `scmRTOS_SYSTIMER_NEST_INTS_ENABLE`       | 0/1     | Enables nested interrupts in the system timer interrupt handler[^15]         |
| `scmRTOS_SYSTEM_TICKS_ENABLE`             | 0/1     | Enables the system timer tick counter                                        |
| `scmRTOS_SYSTIMER_HOOK_ENABLE`            | 0/1     | Enables calling the user-defined function<br>`system_timer_user_hook()` in the system timer interrupt<br> handler. If enabled, this function must be defined in user code |
| `scmRTOS_IDLE_HOOK_ENABLE`                | 0/1     | Enables calling the user-defined function<br> `idle_process_user_hook()` in the `IdleProc` system process.<br> If enabled, this function must be defined in user code |
| `scmRTOS_ISRW_TYPE`                       | `TISRW`<br>`TISRW_SS` | Selects the type of interrupt handler wrapper class for the system<br> timer — regular or with switching to a separate interrupt stack.<br>The `_SS` suffix stands for Separate Stack |
| `scmRTOS_CONTEXT_SWITCH_SCHEME`           | 0/1     | Specifies the context switch method (scheme for transferring<br> control)       |
| `scmRTOS_PRIORITY_ORDER`                  | 0/1     | Defines the priority order in the process map.<br>Value 0 means the highest-priority process corresponds to the<br> least significant bit in the process map (`TProcessMap`); value 1<br> means the highest-priority process corresponds to the most<br> significant (valid) bit |
| `scmRTOS_IDLE_PROCESS_STACK_SIZE`         | N       | Sets the stack size for the background `IdleProc` process                   |
| `scmRTOS_CONTEXT_SWITCH_USER_HOOK_ENABLE` | 0/1     | Enables calling the user-defined hook <br>`context_switch_user_hook()` during context switches. If enabled, the function must be defined<br> in user code |
| `scmRTOS_DEBUG_ENABLE`                    | 0/1     | Enables debugging features                                                  |
| `scmRTOS_PROCESS_RESTART_ENABLE`          | 0/1     | Allows interrupting any process at an arbitrary moment and<br> restarting it from the beginning |

/// Caption
Table 1. Configuration Macros
///

[^15]: If the port supports only one variant, the corresponding macro value is predefined in the port. The same applies to all other macros.
