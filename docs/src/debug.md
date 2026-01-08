# Debugging

## Measuring Process Stack Usage

A common and often difficult question in embedded software development is: what stack size is required for each process to ensure reliable and safe program operation?

In bare-metal programs (without an OS), where all code executes using a single stack, tools exist to estimate required stack memory. These are based on building a call tree and known per-function stack consumption. The compiler can often perform this analysis and report results in the listing file after compilation.

The final estimate is obtained by adding the stack needs of the most demanding interrupt handler to the deepest function call chain.

Unfortunately, this method provides only an approximate estimate. The compiler cannot accurately construct the runtime call tree&nbsp;– particularly for indirect calls (via function pointers or virtual functions), where the actual called function is unknown at compile time. In specific cases where the programmer knows possible indirect targets, manual calculation is possible, but it is inconvenient (requiring recalculation after significant code changes) and error-prone.

In general, compilers are not required to provide such information, and third-party tools face the same limitations and have not gained widespread popularity.

This leaves the developer to choose a stack size balancing memory conservation against sufficient margin to avoid runtime errors. Memory-related bugs (e.g., stack overflow) are notoriously hard to diagnose due to their non-deterministic and highly individual manifestations. In practice, stacks are therefore allocated with some safety margin to account for underestimation.

When using an operating system, the situation worsens: there are multiple stacks—one per configured process—leading to greater RAM pressure and forcing developers to be more conservative with margins.

To address these issues, practical measurement of actual stack consumption per process can be employed. This capability, like other debugging features in **scmRTOS**, is enabled during configuration by setting the macro `scmRTOS_DEBUG_ENABLE` to 1.

The method works as follows: during stack frame initialization, the entire stack area is filled with a known pattern value. Later, to check usage, the stack memory allocated for a process is scanned starting from the end opposite the top-of-stack (TOS), locating the point where the pattern is no longer overwritten. The number of untouched pattern cells indicates the remaining stack slack (unused margin).

Stack pattern filling is performed in the platform-specific `init_stack_frame()` function when debug mode is enabled. The current stack slack for a process can be obtained at any time by calling the process object's function, which returns an integer representing the unused stack size in bytes. Based on this, the developer can adjust stack sizes to eliminate overflow risks.

## Handling Hung Processes

During development, a common scenario arises where the program behaves incorrectly, and indirect evidence points to a specific process not executing. This often occurs when a process is blocked waiting for a service (interprocess communication service object). To diagnose the cause, it is necessary to identify which service the process is waiting on.

For this purpose, **scmRTOS** provides special debugging facilities when debug mode is enabled: upon entering a wait state, the address of the service causing the wait is recorded. When needed, the user can call the process's `waiting_for()` function, which returns a pointer to the service. With the linker map file, this address can be resolved to the service object name.

## Process Profiling

It is often highly valuable to understand the CPU load distribution across processes. This information helps assess algorithm correctness and detect subtle logical errors. Several methods exist for determining relative active execution time&nbsp;– this is known as process profiling.

In **scmRTOS**, profiling is implemented as an extension and is not part of the core OS. The profiler is an extension class providing basic functionality for collecting relative execution time data and processing it. Data collection can be performed in two ways, each with its own advantages and disadvantages:

* Statistical.
* Measurement-based.

### Statistical Method

The statistical method requires no additional resources beyond those provided by the OS. It operates by periodically sampling the kernel variable `CurProcPriority`, which indicates the currently active process. Sampling is conveniently organized, for example, in the system timer interrupt handler: the more CPU time a process consumes, the more frequently it will be sampled as active. The drawback is low accuracy, providing only a qualitative picture.

### Measurement Method

This method eliminates the primary drawback of statistical profiling&nbsp;– low accuracy in determining process load. It works by directly measuring process execution time (hence the name). The user must provide timing resources: typically a hardware timer or, where available, a CPU cycle counter. This is the cost of using this method.

### Usage

To use the profiler in a user project, a time measurement function must be defined and the profiler included in the project. For details, see [the example in the Appendix on Process Profiling](profiler.md).

## Process Names

To improve debugging convenience, processes can be assigned string names. The name is specified in the usual C++ manner via a constructor argument:

```cpp
MainProc main_proc("Main Process");
```

This string argument can always be provided, but name usage is active only in debug configuration.

For access from user code, the `TBaseProcess` class defines the function:

```cpp
const char *name();
```

Usage is straightforward and identical to working with C-strings in C/C++. An example of printing debug information is shown in "Listing 1. Example of Printing Debug Information".

```cpp
01    //-------------------------------------------------------------------------
02    void ProcProfiler::get_results()
03    {
04        print("------------------------------\n");
05        for(uint_fast8_t i = 0; i < OS::PROCESS_COUNT; ++i)
06        {
07        #if scmRTOS_DEBUG_ENABLE == 1
08            printf("#%d | CPU %5.2f | Slack %d | %s\n", i, 
09                   Profiler.get_result(i)/100.0, 
10                   OS::get_proc(i)->stack_slack(), 
11                   OS::get_proc(i)->name() );
12        #endif
13        }
14    }
15    //-------------------------------------------------------------------------
```

/// Caption  
Listing 1. Example of Printing Debug Information  
///

The above code produces output similar to:

```
------------------------------
#0 | CPU 82.52 | Slack 164 | Idle
#1 | CPU  0.00 | Slack 178 | Background
#2 | CPU  0.07 | Slack 387 | GUI
#3 | CPU  0.23 | Slack 259 | Video
#4 | CPU  0.00 | Slack 148 | BiasReg
#5 | CPU 17.09 | Slack 165 | RefFrame
#6 | CPU  0.03 | Slack 204 | TempMon
#7 | CPU  0.00 | Slack 151 | Terminal
#8 | CPU  0.01 | Slack 129 | Test
#9 | CPU  0.01 | Slack 301 | IBoard
```
