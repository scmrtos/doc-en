# Debugging

## Inspecting Process Stack Consumption

There is a question that is often quite difficult to answer definitively: what volume of RAM allocated for the stack is necessary to meet all the program's needs and ensure its correct and safe operation?

In the case of programs running without an OS, where all code executes using a single stack, there are tools for estimating the memory volume allocated to the stack that is necessary for correct operation. They are based on building a function call tree and known information about how much stack each function consumes. The compiler itself can perform this work, placing the results in the listing file after compiling the source file.

To get the final result, one must add the stack requirements of the most consuming interrupt handler to the result of the most consuming function itself.

Unfortunately, the described method only gives an approximate estimate, because the compiler cannot accurately build the call tree of functions that arise in practice — in particular, indirect function calls, which include calls via pointer or virtual function calls, cannot be accounted for, as nothing is known at compile time about which specific function will be called. In specific cases where the programmer knows which functions can be called indirectly, stack consumption calculation can be done manually. But this method is inconvenient — it must be done with every significant program change, and it is error-prone.

In general, the compiler is not obligated to provide such information, and third-party tools performing this work also cannot overcome the difficulties described above, which is why they have not gained popularity.

All this presents the program developer with a choice: what stack size to specify. On one hand, there is a desire to save RAM; on the other, it is necessary to specify a sufficient size to avoid program runtime errors, especially since errors arising from incorrect memory handling are typically very difficult to catch, as their manifestation is always individual and poorly predictable. Therefore, in practice, one has to specify the stack size with some margin, which allows for accounting for errors in underestimating its size.

When using an operating system, the situation is exacerbated because there is not one stack in the program, but their number equals the number of processes specified during OS configuration. This creates a greater RAM deficit and forces the developer to economize even more on memory and specify stack sizes with a smaller margin.

To solve the problems described above, a method of practical measurement of stack consumption by processes can be applied. This capability, like other system debugging capabilities, is enabled in **scmRTOS** during configuration by setting the macro `scmRTOS_DEBUG_ENABLE` to 1.

The essence of the method is to fill the stack space with some predetermined known value (pattern) during the stack frame preparation stage. Then, when checking the result, scan the memory area allocated for the process stack, starting from the end opposite the stack top (TOS), and find the location where the pattern filling ends. The number of cells where the pattern was not overwritten during program execution shows the actual stack size margin for the process.

Filling the stack with the pattern is performed in the platform-dependent function `init_stack_frame()` when debug mode is enabled. Information about the stack margin of a process can be obtained at any time by calling the function ``` for the process object, which returns an integer indicating the desired value. Based on this, the program developer can adjust stack sizes and thereby eliminate errors arising from stack overflow.

## Handling Stuck Processes

During development, a characteristic situation often arises where the program works incorrectly for unclear reasons, and by indirect signs it's easy to determine that a particular process is not working. This usually happens if a process is waiting for some service, and to find the cause of the hang, it's necessary to determine which specific service caused the wait.

To identify the service a process is waiting for, **scmRTOS** includes special debugging tools — in particular, when a process transitions to a waiting state, the address of the service whose call triggered the transition to waiting is stored. If necessary, the user can call the process's `waiting_for()` function, which returns a pointer to the service. Knowing this address, one can always determine the service object's name using the linker map file.

## Process Profiling

Sometimes it is very useful to know the distribution of the processing load among program processes. This information allows assessing the correctness of the program's algorithms and identifying a number of elusive logical errors. Several methods exist for obtaining information about process load by determining the relative time of their active work; this is called process profiling.

In **scmRTOS**, profiling is implemented as an extension and is not part of the core OS. The profiler is an extension class that implements basic functions for collecting information about relative execution time and processing it. Collecting this information can be implemented in two ways, each with its own advantages and disadvantages:

  * Statistical;
  * Measurement-based.

### Statistical Method

The statistical method does not require any additional resources for its operation, except those provided by the operating system. Its working principle is based on periodically sampling the kernel variable `CurProcPriority`, which indicates which process is active at a given moment. Sampling can be conveniently organized, for example, in the system timer interrupt handler — the more CPU time a process occupies, the more often it will be active during sampling. The disadvantage of this method is its low accuracy, allowing only a qualitative picture of what is happening.

### Measurement-based Method

This method is free from the main drawback of statistical profiling — low accuracy in determining process load time. The principle of the measurement-based method is based on measuring the execution time of processes (hence the name). For this, the user must provide means for measuring execution time — this could be one of the MCU's hardware timers or some other means — for example, a processor cycle counter, if available. This is the price for using this method.

### Usage

To use the profiler in a user project, it is necessary to define a time measurement function and connect the profiler to the project. For more details, see the [example in the Process Profiling appendix](profiler.md).

## Process Names

To increase debugging convenience, the ability to assign string names to processes is provided. The name is assigned in the usual way for C++ — through a constructor argument:

```cpp
MainProc main_proc(“Main Process”);
```

This string argument can always be described, but the utilization of the name is available only in the debug configuration.

For access from a user program, the class `TBaseProcess` defines the function:
```cpp
const char *name();
```

Usage is trivial and no different from working with C-strings in C/C++. An example of debug information output can be seen in "Listing 1. Example of Debug Information Output".
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
Listing 1. Example of Debug Information Output
///

The provided code generates the following output:

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
