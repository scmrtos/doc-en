# Processes

----

## General Information and Representation

### The Process Internals

A process in **scmRTOS** is an object of a type derived from the `OS::TBaseProcess` class. The reason why each process requires a separate type (instead of simply making all processes objects of type `OS::TBaseProcess`) is that processes, despite all similarities, still differ&nbsp;–they have different stack sizes and different priority values (which, it should be remembered, are set statically).

Standard C++ templates are used to define process types. This allows for the creation of "compact" process types containing all the necessary internals, including the process stack itself, whose size is different for each process and is specified individually.

```cpp
01    class TBaseProcess
02    {
03       friend class TKernel;
04       friend class TISRW;
05       friend class TISRW_SS;
06       friend class TKernelAgent;
07
08       friend void run();
09
10    public:
11        TBaseProcess( stack_item_t * StackPoolEnd
12                    , TPriority pr
13                    , void (*exec)()
14                #if scmRTOS_DEBUG_ENABLE == 1
15                    , stack_item_t * StackPool
16                #endif
17                    );
18    protected:
19        INLINE void set_unready() { Kernel.set_process_unready(this->Priority); }
20        void init_stack_frame( stack_item_t * StackPoolEnd
21                             , void (*exec)()
22                         #if scmRTOS_DEBUG_ENABLE == 1
23                             , stack_item_t * StackPool
24                         #endif
25                             );
26    public:
27        static void sleep(timeout_t timeout = 0);
28               void wake_up();
29               void force_wake_up();
30        INLINE void start() { force_wake_up(); }
31        INLINE bool is_sleeping() const;
32        INLINE bool is_suspended() const;
33
34    #if scmRTOS_DEBUG_ENABLE == 1
35        INLINE TService * waiting_for() { return WaitingFor; }
36    public:
37               size_t     stack_slack() const;
38    #endif // scmRTOS_DEBUG_ENABLE
39
40    #if scmRTOS_PROCESS_RESTART_ENABLE == 1
41    protected:
42               void reset_controls();
43    #endif
44        //-----------------------------------------------------
45        //
46        //    Data members
47        //
48    protected:
49        stack_item_t *     StackPointer;
50        volatile timeout_t Timeout;
51        const TPriority    Priority;
52    #if scmRTOS_DEBUG_ENABLE == 1
53        TService           * volatile WaitingFor;
54        const stack_item_t * const StackPool;
55    #endif // scmRTOS_DEBUG_ENABLE
56
57    #if scmRTOS_PROCESS_RESTART_ENABLE == 1
58        volatile TProcessMap * WaitingProcessMap;
59    #endif
60
61    #if scmRTOS_SUSPENDED_PROCESS_ENABLE != 0
62        static TProcessMap SuspendedProcessMap;
63    #endif
64    };
```
/// Caption
Listing 1. TBaseProcess
///

### TBaseProcess

The core functionality of a process is defined in the base class `OS::TBaseProcess`. As mentioned above, the actual processes are derived from it using the `OS::process<>` template. This method is used to avoid duplicating identical code in template instances[^1] during their implementation.

[^1]: The term "instance" is often used in jargon.

Therefore, the template itself declares only what pertains to entities that differ between processes—stacks and the process execution function (`exec()`). The source code of the `OS::TBaseProcess` class is presented[^2]—see "Listing 1. TBaseProcess".

[^2]: Actually, there are two variants of this class—the standard one (shown here) and one with a separate stack for return addresses. The latter is not included here for brevity, as it contains no fundamental differences relevant for understanding or exposition.

Despite the seemingly extensive definition of this class, it is actually very small and simple. Its representation contains only three data members—the stack pointer (49), the timeout tick counter (50), and the priority value (51). The remaining data members are auxiliary and are present only when additional functionality is enabled—the ability to interrupt a process at any moment with subsequent restart, as well as debugging facilities[^3].

[^3]: The same applies to the rest of the code—much of the class definition is occupied by the description of these auxiliary capabilities.

The class interface provides the following functions:

*   `sleep(timeout_t timeout = 0)`. Transitions the process to a "sleep" state: the argument value is assigned to the internal timeout counter variable, the process is removed from the map of ready processes, and the scheduler is called, which will transfer control to the next ready process.
*   `wake_up()`. Wakes the process from the "sleep" state. The process is transitioned to the ready state only if it was in a timeout-based waiting state for an event; if this process has a priority higher than the current one, it immediately gains control.
*   `force_wake_up()`. Wakes the process from the "sleep" state. The process is always transitioned to the ready state. If this process has a priority higher than the current one, it immediately gains control. This function must be used with extreme caution, as incorrect use can lead to improper (unpredictable) program behavior.
*   `is_sleeping()`. Checks if the process is in a "sleep" state, i.e., in a timeout-based waiting state for an event.
*   `is_suspended()`. Checks if the process is in an inactive state.


### Stack

A process stack is a contiguous area of RAM used for storing the process data, as well as for saving the process context and return addresses from functions and interrupts.

Due to the specifics of some architectures, two separate stacks may be used — one for data, another for return addresses. **scmRTOS** supports this capability, allowing for the placement of two RAM areas — two stacks — within each process object. The size of each can be specified individually based on the requirements of the application. Support for two stacks is enabled using the `SEPARATE_RETURN_STACK` macro defined in the `os_target.h` file.

Inside the protected section, the very important function `init_stack_frame()` is declared. It is responsible for forming the stack frame. The fact is that the startup of process executable functions does not occur in the same way as for regular functions — process executable functions are not called in the traditional manner. Control is transferred to them in the same way as during context switching between processes. Therefore, starting a process's executable function occurs by restoring that process's context from the stack, followed by a jump to the address stored in the stack at the location of the saved process interrupt return address.

To make such a startup possible, the process stack must be prepared accordingly — by initializing memory cells in the stack at specific addresses with the necessary values. In other words, the process stack's contents must be as if the process had been preempted earlier (with its context, of course, saved). The specific actions for preparing the stack frame are individual for each platform, hence the implementation of the `init_stack_frame()` function is placed at the operating system porting level.

### Timeouts

Each process has a special variable `Timeout` for controlling the process behavior during event waits with timeouts or during "sleep". Essentially, this variable is a counter for system timer ticks. If its value is not zero, it is decremented and compared to zero within the system timer interrupt handler. Upon reaching zero, the process owning this variable is transitioned to the ready state.

Thus, if a process is in a "sleep" state with a timeout (i.e., placed into the not-ready state by calling the `sleep(timeout)` function with a non-zero argument), then after an interval equal to the specified number of system timer ticks[^4], the process will be "awoken"[^5] within the system timer interrupt handler.

[^4]: Strictly speaking, not exactly equal to the number of system timer ticks, but with an accuracy up to a fraction of this period, which depends on the moment the `sleep` function is called relative to the moment the system timer interrupt occurs.

[^5]: I.e., transitioned to the ready state.

A similar situation occurs when calling a service function that involves waiting for an event with a timeout. In this case, the process will be transitioned to the ready state either upon the occurrence of the event it is waiting for (which triggered the service function call) or upon timeout expiration. The value returned by the service function unambiguously indicates the source of the process's "awakening", allowing the user program to easily decide on further actions in the given situation.

### Priorities

Each process also has a data field containing the process priority. This field serves as the process identifier for manipulations with processes and their representations. In particular, the process priority is the index into the table of pointers to processes located within the kernel, where the address of each process is written during registration.

Priorities are unique — there cannot be two processes with the same priority. The internal representation of a priority is an integer-type variable. For safety when assigning priorities, a special enumerated type `TPriority` is used.

<a name="process-sleep"></a>
### The sleep() Function

This function serves to transition the current process from the active state to the inactive state. If the function is called with an argument equal to 0 (or without specifying an argument — the function is declared with a default argument equal to 0), the process will go into "sleep" until it is awakened, for example, by another process using the `TBaseProcess::force_wake_up()` function. If the function is called with an argument, the process will "sleep" for the specified number of system timer ticks, after which it will be "awoken", i.e., placed into the ready state. In this case, the "sleep" can also be interrupted by another process or an interrupt handler using the `TBaseProcess::wake_up()` or `TBaseProcess::force_wake_up()` functions.

----

## Process Creation and Usage

### Defining a Process Type

To create a process, you need to define its type and declare an object of that type.

The type of a specific process is described using the `OS::process` template: see "Listing 2. Process Template".

```cpp
01    template<TPriority pr, size_t stack_size>               
02    class process : public TBaseProcess                     
03    {                                                       
04    public:                                                 
05        INLINE_PROCESS_CTOR process();                      
06                                                            
07        OS_PROCESS static void exec();                      
08                                                            
09    #if scmRTOS_PROCESS_RESTART_ENABLE == 1                 
10        INLINE void terminate();                            
11    #endif                                                  
12                                                            
13    private:                                                
14        stack_item_t Stack[stack_size/sizeof(stack_item_t)];
15    };                                                      
```
/// Caption
Listing 2. Process Template
///

As you can see, two entities are added to what the base class provides:

  * the process stack `Stack` with size `stack_size`. The size is specified in bytes;
  * the static function `exec()`, which is the actual function containing the user code of the process.

### Declaring a Process Object and Its Usage

Now it's enough to declare an object of this type, which will be the process itself, and to define the process function `exec()`.

```cpp

typedef OS::process<OS::prn, 100> Slon;

Slon slon;
```
where n is the priority number.

["Listing 1. Process Executable Function"](overview.md#process-exec) illustrates an example of a typical process function.

Using a process mainly consists of writing user code inside the process function. In doing so, as already mentioned, a few simple rules should be followed:

  * You must ensure that the program's control flow does not leave the process function. Otherwise, due to the fact that this function was not called in the usual way, exiting from it will cause the control flow to go, roughly speaking, to undefined addresses, leading to undefined program behavior (although in practice, the behavior is usually quite definite&nbsp;– the program does not work!);
  * Use the `TBaseProcess::wake_up()` function with caution and attention, and `TBaseProcess::force_wake_up()`&nbsp;– with extra caution, as careless use can lead to untimely "awakening" of a sleeping (suspended) process, which may cause collisions in inter-process communication.

### Starting a Process in an Inactive State

Sometimes there is a need for the process executable function to start not immediately after system startup, but upon a specific signal. For example, there are several processes that should start their work only after initializing/configuring some (possibly external to the MCU) equipment; otherwise, unpleasant consequences may arise due to incorrect actions towards that equipment.

In this situation, some dispatching will be required&nbsp;– the processes must somehow organize their work so as not to disrupt the logic of interaction with this equipment&nbsp;– for example, all processes except one (the dispatcher) start by waiting for an event (work start), which will be signaled to them by the dispatcher process.

The dispatcher process performs all necessary preparatory work and then announces the start of work to the waiting processes. The described approach would require adding corresponding code manually in each process waiting for the start, which clutters the code, adds work, and is error-prone.

Furthermore, there may be other situations where it is required that a process does not start its work immediately. To provide the described functionality, a process has the ability to start in a so-called inactive state. Such a process is no different from any other except that its tag is absent in the map of processes ready for execution (`ReadyProcessMap`).

The declaration of such a process looks like this[^6]:
```cpp
typedef OS::process<OS::pr1, 300, OS::pssSuspended> Proc2;
...
Proc2 proc2;
```

Later, to start the work of this process, the launching code must call the `force_wake_up()` function:

```cpp
Proc2.force_wake_up();
```
[^6]: The ss prefix in this example stands for Start State

---

<a name="process-restart"></a>
## Process Restart

A situation may arise when it is necessary to interrupt a process from the outside and start its execution from the beginning. For example, a process performs lengthy calculations, and it turns out that the results of these calculations are no longer needed at some point, and a new calculation cycle with new data must be started. This can be done by terminating the process with the possibility of subsequently restarting it from the beginning.

To implement the above, the OS provides the user with two functions:

  * `OS::process::terminate()`;
  * `OS::TBaseProcess::start()`.

The `terminate()` function is intended to be called from outside the process being stopped. Inside it, all resources associated with this process are reset to their initial state, and the process is transitioned to the not-ready state. If the process was waiting for a service, its tag is removed from that service's map of waiting processes.

Process startup is performed separately&nbsp;– so that the user has the opportunity to do it at a moment they deem appropriate&nbsp;– and is carried out using the `start()` function, which simply transitions the process to the ready state. The process will start working according to the sequence determined by its priority and the OS load.

For the process interruption and restart to work correctly, this functionality must be enabled during configuration&nbsp;– the value of the `scmRTOS_PROCESS_RESTART_ENABLE` macro must be set to 1.

