# Operating System Overview

## General Information

**scmRTOS** is a real-time operating system with priority-based preemptive multitasking. The OS supports up to 32 processes (including the system **IdleProc** process, meaning up to 31 user processes), each with a unique priority. All processes are static, i.e., their number is determined at the project build stage and they cannot be added or removed during runtime.

The decision to forgo dynamic process creation is driven by the need to conserve resources, which are very limited in single-chip MCUs. Dynamic process deletion is also not implemented because it offers little benefit—the program memory used by a process is not freed, and RAM intended for future use would require the ability to be allocated and deallocated using a memory manager. Such a manager is a complex component in itself, requires significant resources, and is typically not used in single-chip MCU projects[^1].

In the current version, process priorities are also static. Each process receives its priority at the project build stage, and this priority cannot be changed during program execution. This approach is similarly motivated by the goal of making the system as lightweight as possible in terms of resource requirements and runtime overhead. Changing priorities during system operation is a highly non-trivial mechanism. For it to work correctly, it requires analyzing the state of the entire system (kernel, services) followed by modifying kernel components and other OS parts (semaphores, event flags, etc.). This inevitably leads to prolonged periods of operation with interrupts disabled and, consequently, significantly degrades the system's real-time performance.

[^1]: This refers to the standard memory manager provided with development tools. There are situations where a program needs to store data between function calls (i.e., using automatic storage duration—on the stack or in CPU registers—is not suitable), and the total amount of such data is unknown at compile time—its appearance and lifetime are determined by events occurring during program execution. For storing such data, the heap is the most appropriate place. Managing this heap is typically the responsibility of a memory manager. Therefore, some applications cannot do without such a component. However, given the resource consumption of a standard memory manager, its use often becomes impractical. In these situations, a specialized memory manager, designed specifically to meet the application's requirements optimally, is frequently employed. Considering the above, it becomes clear that creating a universal memory manager equally well-suited to the diverse needs of various projects is hardly realistic. This is why **scmRTOS** does not include a built-in memory manager.

## OS Structure

The system consists of three main components: the Kernel, processes, and inter-process communication (IPC) facilities.

### Kernel

The kernel performs the following functions:

*   Process organization and management.
*   Scheduling at both the process and interrupt levels.
*   Support for inter-process communication.
*   System time support (system timer).
*   Support for extensions.

For more details on the kernel's structure, composition, functions, and mechanisms, see the [OS Kernel](kernel.md) section.

### Processes

Processes provide the ability to create a separate (asynchronous relative to others) program execution flow. For this purpose, each process implements a function that must contain an infinite loop, which serves as the process's main loop&nbsp;– see "Listing 1. Process Execution Function" for an example.

<a name="process-exec"></a>
```cpp
1    template<> void slon_proc::exec()
2    {
3        ... // Declarations
4        ... // Init process’s data
5        for(;;)
6        {
7            ... // process’s main loop
8        }
9    }
```

/// Caption
Listing 1. Process Execution Function
///

Upon system startup, control is transferred to the process function. At its beginning, you can place declarations for the data it uses (3) and initialization code (4), followed by the process's main loop (5)-(8). The user code must be written to prevent exiting the process function. For example, once it enters the main loop, it should not leave it (the primary approach). Alternatively, if it does exit the main loop, it must enter another loop (even an empty one) or go into an infinite "sleep" by calling the `sleep()`[^2] function without parameters (or with parameter "0"). For more details, see the [`sleep()` Function](processes.md#process-sleep). The process code must also not contain `return` statements to exit the function.

[^2]: In this case, no other process should "wake up" this sleeping process before it exits, otherwise undefined behavior will occur and the system will likely crash. The only safe action that can be applied to a process in this situation is to terminate it (with the possibility of restarting it from the beginning later). See [Process Restart](processes.md#process-restart).

### Inter-Process Communication

Since processes in the system execute in parallel and asynchronously relative to each other, simply using global data for communication between them is incorrect and unsafe. While one process is accessing an object (which could be a variable of a built-in type, an array, a structure, a class object, etc.), its execution can be interrupted by another (higher-priority) process that also accesses the same object. Due to the non-atomic nature of access operations (read/write), the second process can either disrupt the correctness of the first process's actions or simply read incorrect data.

To prevent such situations, special measures must be taken: access must be performed within so-called Critical Sections (where control transfer between processes is prohibited), or specialized inter-process communication (IPC) facilities must be used. The following IPC facilities are available in **scmRTOS**:

*   Event Flags (`OS::TEventFlag`);
*   Mutual Exclusion Semaphores (`OS::TMutex`);
*   Data Channels in the form of a queue for bytes or objects of arbitrary type (`OS::channel`);
*   Messages (`OS::message`).

The developer must decide which facility (or combination thereof) to use in each specific case, based on task requirements, available resources, and personal preference.

Starting with **scmRTOS v4**, inter-process communication facilities (services) are built upon a common specialized base class, `TService`. This class provides all the necessary foundational tools for implementing service classes/templates. Its interface is documented and intended for users to extend the set of services themselves. If needed, a user can design and implement their own IPC facility that best meets the requirements of a specific target project.

## Software Model

### Composition and Organization

The **scmRTOS** source code in any project consists of three parts: the common (core), platform-dependent (target), and project-dependent (project) parts.

The **common part** contains declarations and definitions for kernel functions, processes, system services, as well as a small support library containing some useful code, parts of which are used directly by the OS.

The **platform-dependent part** includes declarations and definitions responsible for implementing functions inherent to the specific target platform, language extensions for the compiler used, etc. This part encompasses the assembly code for context switching and system startup, the function for setting up the stack frame structure, the definition of a critical section wrapper class, the interrupt handler for the hardware timer used as the system timer on that platform, and other platform-dependent behaviors.

The **project-dependent part** consists of three header files containing configuration macro definitions, includes for extensions, and code necessary for fine-tuning the operating system for a specific target project. This includes, in particular, type alias definitions for setting the bit width of timeout variables, selecting the source for the context switch interrupt, and other tools required for optimal system operation.

The recommended placement of system source files is as follows:
*   Common part&nbsp;– in a separate `core` directory.
*   Platform-dependent part&nbsp;– in its own `<target>` directory, where `target` is the name of the system's target port.
*   Project-dependent part&nbsp;– directly within the project's source files.

This structure is suggested for ease of project storage, transfer, and maintenance, as well as for a simpler and safer system update process when migrating to new versions.

The source code of the common part is contained in eight files:

*   `scmRTOS.h`&nbsp;– The main header file; it includes the entire hierarchy of system header files.
*   `os_kernel.h`&nbsp;– Core declarations and type definitions for the OS kernel.
*   `os_kernel.cpp`&nbsp;– Object declarations and function definitions for the kernel.
*   `scmRTOS_defs.h`&nbsp;– Auxiliary declarations and macros.
*   `os_services.h`&nbsp;– Type and service template definitions.
*   `os_services.cpp`&nbsp;– Service function definitions.
*   `usrlib.h`&nbsp;– Type and template definitions for the support library.
*   `usrlib.cpp`&nbsp;– Support library function definitions.

As evident from the list above, **scmRTOS** also includes a small support library containing code used by the OS facilities[^3]. Since this library is not essentially part of the OS itself, its examination will not be covered in detail in this document.

[^3]: Notably, the ring buffer class/template.

The source code of the platform-dependent part is located in three files:

*   `os_target.h`&nbsp;– Platform-dependent declarations and macros.
*   `os_target_asm.ext`[^4]&nbsp;– Low-level code, context switching functions, OS startup.
*   `os_target.cpp`&nbsp;– Definitions for the process stack frame initialization function, the interrupt handler function for the timer used as the system timer, and the root function of the background (idle) process.

[^4]: The assembly file extension for the target processor.

The project-dependent part consists of three header files:

*   `scmRTOS_config.h`&nbsp;– Configuration macros and aliases for some types, particularly the type defining the bit width of timeout objects.
*   `scmRTOS_target_cfg.h`&nbsp;– Code for tuning OS mechanisms to the needs of a specific project. This can include, for example, assigning the interrupt vector for the hardware timer interrupt handler chosen as the system timer, macros for controlling the system timer, defining the function for activating the context switch interrupt, etc.
*   `scmRTOS_extensions.h`&nbsp;– Manages the inclusion of extensions. For more details, see [TKernelAgent and Extensions](kernel.md#kernel-agent).

Понял, исправляю. Уберу выделение жирным для названий в `` и буду строже следовать оригинальному форматированию.

### Internal Structure

Everything related to **scmRTOS**, except for a few functions implemented in assembly and having the `extern "C"` linkage specification, is placed inside the `OS` namespace — this method implements a separate namespace for the operating system's components.

The following classes are declared within this namespace[^5]:

*   `TKernel`. Since the kernel in the system can be represented by only one instance, only one object of this class exists. The user must not create objects of this class;
*   `TBaseProcess`. Implements the object type that serves as the basis for constructing the `process` template, upon which any (user or system) OS process is implemented;
*   `process`. A template upon which the type of any OS process is created.
*   `TISRW`. This is a wrapper class to simplify and automate the procedure for creating interrupt handler code. Its constructor performs actions upon entering an interrupt handler, and its destructor performs the corresponding actions upon exit.
*   `TKernelAgent`. A special service class intended to provide access to the necessary kernel resources for extending OS capabilities. The `TService` class, which serves as the base for all inter-process communication facilities, as well as the profiler class, are built upon this class.

[^5]: Almost all OS classes are declared as friends (`friend`) of each other. This is done to provide access for the OS components to the internal representations of other components without exposing the interface externally, thereby preventing user code from directly using internal OS variables and mechanisms, which enhances usage safety.

The list of service classes includes:

*   `TService`. A base class for building all types and templates of inter-process communication facilities. Contains common functionality and defines the Application Programming Interface (API) for all descendant types. Serves as the basis for extending the set of IPC facilities.
*   `TEventFlag`. Designed for inter-program interactions via a binary semaphore (event flag);
*   `TMutex`. A binary semaphore designed for organizing mutual exclusion of access to shared resources;
*   `message`. A template for creating message objects. A message is "similar" to an event flag but can additionally contain an object of an arbitrary type (usually a structure) representing the message body;
*   `channel`. A template for creating a data channel for an arbitrary type. Serves as the basis for building message queues.

As can be seen from the list above, counting semaphores are absent. The reason for this is that, despite all desire, no acute necessity for them could be identified. Resources that need to be controlled using counting semaphores are in acute shortage in single-chip MCUs, primarily RAM. Situations where it is still necessary to control the available quantity are handled using objects created based on the `OS::channel` template, which internally already implements the corresponding mechanism in one form or another. If such a service is needed, the user can independently add it to the base set by creating their own implementation as an extension; see [TKernelAgent and Extensions](kernel.md#kernel-agent).

**scmRTOS** provides the user with several control functions:

*   `run()`. Designed for starting the OS. When this function is called, the actual operation of the operating system begins — control is transferred to the processes, whose work and mutual interaction are determined by the user program. After transferring control to the OS kernel code, the function does not regain it (control) and, consequently, a return from the function is not provided;
*   `lock_system_timer()`. Blocks interrupts from the system timer. Since the selection and servicing of the system timer hardware fall under the project's competence, the user must define the content of this function. The same applies to the paired function `unlock_system_timer()`;
*   `unlock_system_timer()`. Unblocks interrupts from the system timer;
*   `get_tick_count()`. Returns the number of system timer ticks. The system timer tick counter must be enabled when configuring the system;
*   `get_proc()`. Returns a pointer to a constant process object based on the index passed to the function as an argument. The index is essentially the value of the process's priority.

### Critical Sections

Due to the preemptive nature of process execution, any process can be interrupted at an arbitrary point in time. On the other hand, there are a number of cases[^6] where it is necessary to eliminate the possibility of interrupting a process during the execution of a specific code fragment. This is achieved by disabling task switching[^7] for the duration of that fragment's execution. In other words, this fragment becomes a non-interruptible section.

[^6]: For example, accessing OS kernel variables or the internal state of inter-process communication facilities.
[^7]: In the current implementation of **scmRTOS**, this is achieved by disabling interrupts globally.

In OS terminology, such a section is called a **critical section**. To simplify the creation of a critical section, a special wrapper class, `TCritSect`, is provided. The constructor of this class saves the state of the processor resource that controls the global enabling/disabling of interrupts and then disables interrupts. The destructor restores this processor resource to the state it was in before the interrupts were disabled.

Thus, if interrupts were already disabled, they will remain disabled. If they were enabled, they will be re-enabled. The implementation of this class is platform-dependent, so its definition is located in the corresponding `os_target.h` file.
Using `TCritSect` is straightforward: at the point that marks the beginning of the critical section, simply declare an object of this type. From the point of declaration until the end of the block, interrupts will be disabled[^8].

[^8]: Upon exiting the block, the destructor is automatically called, which restores the state that existed before entering the critical section. This method eliminates the possibility of "forgetting" to re-enable interrupts when exiting the critical section.

### Built-in Type Aliases

To simplify working with the source code and to enhance portability, the following type aliases are defined:

*   `TProcessMap`&nbsp;– A type for defining a variable that functions as a **process map**. Its size depends on the number of processes in the system. Each process corresponds to a unique tag—a mask containing only one non-zero bit, positioned according to the priority of that process. The process with the highest priority corresponds to the least significant bit (position 0)[^9]. With fewer than 8 user processes, the process map size is 8 bits. For 8 to 15 user processes, the size is 16 bits. For 16 or more user processes, the size is 32 bits.
*   `stack_item_t`&nbsp;– The type of a **stack element**. This depends on the target architecture. For example, on the 8-bit **AVR**, this type is defined as `uint8_t`; on the 16-bit **MSP430**, as `uint16_t`; and on 32-bit platforms, typically as `uint32_t`.

### Using the OS

As mentioned earlier, static mechanisms were used wherever possible to achieve maximum efficiency. This means that all functionality is determined at the compilation stage.

This applies primarily to processes. Before using each process, its type must be defined[^10]. This definition specifies the process type's name, its priority, and the amount of RAM allocated for its [stack](). For example:

[^9]: This order is the default. If `scmRTOS_PRIORITY_ORDER` is defined as 1, the order of bits in the process map is reversed—i.e., the most significant bit corresponds to the highest priority process, and the least significant bit corresponds to the lowest priority process. The reversed priority order can be useful for processors with hardware support for finding the first non-zero bit in a binary word—for example, for processors of the **Blackfin** family.
[^10]: Each process is an object of a distinct type (class), derived from the common base class `TBaseProcess`.

```cpp
OS::process<OS::pr2, 200> MainProc;
```

Here, a process with priority `pr2` and a stack size of 200 bytes is defined. This declaration might seem somewhat inconvenient due to its verbosity, because whenever you need to refer to the process type, you must write the full declaration—for example, when defining the process's execution function[^11]:

[^11]: The execution function of a specific process is technically a full specialization of the member function template `OS::process::exec()`. Therefore, its definition uses the syntax for defining a specialization: `template<>`.

```cpp
template<> void OS::process<OS::pr2, 200>::exec() { ... }
```

This is because the exact type is the expression
```cpp
OS::process<OS::pr2, 200>
```

A similar situation will arise in other cases where you need to refer to the process type. To eliminate this inconvenience, you can use type aliases introduced via `typedef`. This is the recommended coding style: first define type aliases for the processes (preferably in a single place, such as a header file, to have an immediate overview of how many processes are in the project and what they are), and then declare the actual process objects in the appropriate source files. With this approach, the example above looks like this[^12]:

[^12]: It is recommended to declare the prototype for the specialization of the process's execution function before the first use of the template instance—this allows the compiler to see that a full specialization of the function exists for this instance, so there is no need to attempt generating the general implementation of that template function. In some cases, this helps avoid compilation errors.

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

There is nothing special about this sequence of actions—it is the standard way of describing a type alias and creating an object of that type, common in the C and C++ programming languages.

!!! warning "**IMPORTANT NOTE**"

    The number of processes must be specified when configuring the system. This number must exactly match the number of processes described in the project; otherwise, the system will not work. Please note that a special enumerated type, `TPriority`, is introduced for specifying priorities. It describes the allowable priority values[^13].

    Furthermore, the priorities of all processes must be consecutive, without gaps. For example, if there are 4 processes in the system, their priorities must be `pr0`, `pr1`, `pr2`, `pr3`. Duplicate priority values are also not allowed, meaning each process must have a unique priority value. For instance, if there are 4 user processes in the system (i.e., 5 processes in total, including the `IdleProc` system process), the priority values must be `pr0`, `pr1`, `pr2`, `pr3` (`prIDLE` for `IdleProc`), where `pr0` is the highest priority process and `pr3` is the lowest priority among the user processes. The absolute lowest priority process is `IdleProc`. This process always exists in the system and does not need to be described. It is this process that gains control when all user processes are in an inactive state.

    The compiler does not monitor gaps in process priority numbering or the uniqueness of process priority values. Adhering to the principle of separate compilation, there is no efficient way to automate configuration integrity checks using language features alone.

    Currently, there is a special tool that performs all the work of checking configuration integrity. The utility is called **scmIC** (IC&nbsp;– Integrity Checker) and can detect the vast majority of typical OS configuration errors.

[^13]: This is done to enhance safety—you cannot simply specify any integer value; only the values described in `TPriority` are valid. The values described in `TPriority` are linked to the number of processes specified by the configuration macro `scmRTOS_PROCESS_COUNT`. Thus, you can only choose from a limited set. Process priority values look like: `pr0`, `pr1`, etc., where the number denotes the priority level. The system process `IdleProc` has a separate priority designation, `prIDLE`.

As mentioned, it is convenient to place the definitions of process types in a header file to make any process easily visible in another compilation unit.

An example of typical process usage can be found in "Listing 2. Defining Process Types in a Header File" and "Listing 3. Declaring Processes in a Source File and Starting the OS".

```cpp
 1    //------------------------------------
 2    //
 3    //      Process types definition
 4    //
 5    //
 6    typedef OS::process<OS::pr0, 200> TUARTDrv;
 7    typedef OS::process<OS::pr1, 100> TLCDProc;
 8    typedef OS::process<OS::pr2, 200> TMainProc;
 9    typedef OS::process<OS::pr3, 200> TFPGA_Proc;
10    //-------------------------------------
```
/// Caption
Listing 2. Defining Process Types in a Header File
///

```cpp
 1    //-------------------------------------
 2    //
 3    //  Processes declarations
 4    //
 5    //
 6    TUartDrv   UartDrv;
 7    TLCDProc   LCDProc;
 8    TMainProc  MainProc;
 9    TFPGAProc  FPGAProc;
10    //-------------------------------------
11
12    //-------------------------------------
13    void main()
14    {
15        ... // system timer and other stuff initialization
16        OS::run();
17    }
18    //-------------------------------------
```
/// Caption
Listing 3. Declaring Processes in a Source File and Starting the OS
///

As mentioned earlier, each process has an execution function. When using the scheme described above, the process execution function is named `exec` and looks as shown in "Listing 1. Process Execution Function".

Configuration information is specified in a special header file, `scmRTOS_config.h`. The list and values[^14] of the configuration macros are shown in "Table 1. Configuration Macros".

[^14]: The table provides example values. In each project, values are set individually based on the project's requirements.

| Name | Value | Description |
| :--- | :--- | :--- |
| `scmRTOS_PROCESS_COUNT` | n | Number of processes in the system |
| `scmRTOS_SYSTIMER_NEST_INTS_ENABLE` | 0/1 | Enables nested interrupts within the system timer interrupt handler[^15] |
| `scmRTOS_SYSTEM_TICKS_ENABLE` | 0/1 | Enables the use of the system timer tick counter |
| `scmRTOS_SYSTIMER_HOOK_ENABLE` | 0/1 | Enables the call to the `system_timer_user_hook()` function within the system timer interrupt handler. In this case, the specified function must be defined in the user code |
| `scmRTOS_IDLE_HOOK_ENABLE` | 0/1 | Enables the call to the `idle_process_user_hook()` function within the system `IdleProc` process. In this case, the specified function must be defined in the user code |
| `scmRTOS_ISRW_TYPE` | `TISRW`<br>`TISRW_SS` | Selects the type of wrapper class for the system timer interrupt handler — regular or with a switch to a separate interrupt stack. The suffix `_SS` stands for Separate Stack |
| `scmRTOS_CONTEXT_SWITCH_SCHEME` | 0/1 | Defines the method for context switching (transfer of control) |
| `scmRTOS_PRIORITY_ORDER` | 0/1 | Defines the priority order in the process map. A value of 0 corresponds to the case where the highest priority process maps to the least significant bit in the process map (`TProcessMap`). A value of 1 corresponds to the case where the highest priority process maps to the most significant (relevant) bit in the process map |
| `scmRTOS_IDLE_PROCESS_STACK_SIZE` | N | Defines the stack size for the `IdleProc` background process |
| `scmRTOS_CONTEXT_SWITCH_USER_HOOK_ENABLE` | 0/1 | Enables the call to the user hook `context_switch_user_hook()` during context switching. In this case, the function must be defined in the user code |
| `scmRTOS_DEBUG_ENABLE` | 0/1 | Enables debugging facilities |
| `scmRTOS_PROCESS_RESTART_ENABLE` | 0/1 | Allows interrupting the execution of any process at an arbitrary moment and restarting that process |

/// Caption
Table 1. Configuration Macros
///

[^15]: If the port supports only one variant, the corresponding macro value is defined within the port itself. The same applies to all other macros.
