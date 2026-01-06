## Terms and Abbreviations

#### C

A general-purpose, low-level procedural programming language.

#### C++

A general-purpose programming language supporting procedural, object-based, and object-oriented programming paradigms.

#### Critical Section

A code fragment during whose execution control transfer is prohibited. In scmRTOS, this is currently implemented in the simplest way by globally disabling interrupts.

#### EC++

Embedded C++, a subset of C++. It does not support namespaces, templates, multiple inheritance, RTTI, exception handling, or the new explicit type conversion syntax.

#### Idle Process (Background Process)

A system process that gains control when all user processes are in a state of waiting for events. This process cannot transition to a waiting state and may execute a user hook if enabled during configuration.

#### Inter-Process Communication (IPC) Services

Objects and/or OS extensions designed for safe interaction (synchronization of work and data exchange) between different processes, as well as for organizing event-driven program execution based on events occurring in interrupts and processes.

#### Interrupt Stack

A specially allocated RAM area intended for use as a stack during the execution of interrupt handler code. If an interrupt stack is used in the program, the processor's stack pointer is switched to the interrupt stack upon entering an interrupt handler and switched back to the process stack upon exit.

#### ISR

Interrupt Service Routine – an interrupt handler.

#### Kernel

The most important and central part of the operating system, performing functions related to organizing processes, scheduling their execution, supporting inter-process communication, system time, and OS extensions.

#### MCU

Microcontroller.

#### Operating System Process

An object that implements the execution of a complete, independent program fragment asynchronous to others, including support for control transfer both at the process level and at the interrupt level.

#### OS Extensions

Software objects that extend the functionality of the operating system but are not part of the core OS. An example of an extension is the system process activity profiler.

#### OS

Operating System.

#### OS Configuration

The set of macros, types, other definitions, and declarations that specify the quantitative and qualitative characteristics and properties of the operating system in a specific project. Configuration is performed by defining the contents of special header configuration files, as well as by some user code executed before OS startup.

#### OS Port

The combination of common and platform-dependent OS code tailored to a specific software and hardware platform.

#### Preemption

The set of actions performed by operating system components aimed at forcibly transferring control from one process to another.

#### Process Context

The software and hardware environment of the executing code, including processor registers, stack pointers, and other resources necessary for program execution. Since control transfer from one process to another in a preemptive OS can occur at an unpredictable moment, the process context must be saved until the next time this process gains control. Because each process executes independently and asynchronously relative to other processes, to ensure the correct operation of a preemptive OS, each process must have its own context.

#### Process Executable Function

A static member function of the process class that implements an independent, asynchronous program execution flow in the form of an infinite loop.

#### Process Map

An operating system object containing one or more process tags. Physically implemented based on an integer variable. Each bit in the process map corresponds to one process and uniquely maps to the process priority.

#### Process Priority

A property of a process (an integer-type object) that determines the order of process selection during operations in the scheduler and other OS components. Serves as a unique process identifier.

#### Process Stack

A memory area in the form of an array that is a data member of the process object, used as a stack in the process's executable function. Also serves as the location where the process context is saved during control transfer.

#### Process Tag

A binary number mask containing only one non-zero bit, whose position is uniquely related to the process priority number. Like the process priority, the tag is a unique identifier but has a different representation than the process priority. Each representation (priority or tag) is used where it is more appropriate from the perspective of program efficiency.

#### Profiler

An object that measures, by one means or another, the distribution of processor time among processes and has facilities for providing this information to the user.

#### RAM

Random Access Memory.

#### Ring Buffer

A data object representing a queue. It has two data ports (access functions) – an input for writing and an output for reading. Implemented using an array and two indices (pointers) denoting the start and end of the queue. Upon reaching the physical end of the array, writing/reading starts from the beginning again, meaning the indices move in a ring, hence the object's name.

#### RTOS

Real-Time Operating System.

#### Scheduler

A core component of the OS kernel responsible for managing the order of process execution.

#### Stack Frame

Represents a set of data placed in the process stack exactly as it is when the process context is saved during control transfer.

#### System Timer

A hardware timer of the target processor selected as the source for generating interrupts at a specified period, as well as the OS function called from the timer's ISR that implements the logic for handling process timeouts.

#### User Hook

A function called from the OS code, whose body must be defined by the user. This allows user-defined code to be executed directly from within the operating system's internal functions without modifying its code.
To avoid forcing the user to define the bodies of hooks they do not use, the hook is called only if enabled during configuration. That is, if a user wishes to use a particular hook, they must enable its use and define its body.

#### Timeout

A time interval, specified by an integer-type object, used for organizing conditional or unconditional event waiting by processes.

#### TOS

Top Of Stack. The address of the stack element pointed to by the processor's hardware stack pointer.
