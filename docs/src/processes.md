# Processes
----

## General Information and<br> Internal Representation

### The Process Concept

In **scmRTOS**, a process is an object of a type derived from the class `OS::TBaseProcess`. The reason each process requires its own distinct type—rather than simply creating all processes as objects of `OS::TBaseProcess`—is that, despite their similarities, processes differ in key aspects: they have different stack sizes and different priority values (which, it should be remembered, are assigned statically).

To define process types, the standard C++ feature—templates—is used. This approach yields compact process types that contain all necessary internal data, including the process stack itself, which varies in size across processes and is specified individually.

### TBaseProcess

The core functionality of a process is defined in the base class `OS::TBaseProcess`, from which actual processes are derived using the `OS::process<>` template, as mentioned earlier. This approach is chosen to avoid duplicating identical code across template instantiations[^1].

[^1]: In programming slang, these are often called instances.

Therefore, the template itself declares only those elements that differ between processes&nbsp;– the stacks and the process executable function (`exec()`). The source code for the class `OS::TBaseProcess` is presented[^2], see "Listing 1. TBaseProcess".

[^2]: In reality, there are two variants of this class: the standard one (shown here) and a version with a separate return-address stack. The latter is omitted for brevity, as it introduces no fundamental differences relevant to understanding the concepts.

```cpp
01    class TBaseProcess                                                             
02    {                                                                              
03        friend class TKernel;                                                      
04        friend class TISRW;                                                        
05        friend class TISRW_SS;                                                     
06        friend class TKernelAgent;                                                 
07                                                                                   
08        friend void run();                                                         
09                                                                                   
10    public:                                                                        
11        TBaseProcess( stack_item_t * StackPoolEnd                                  
12                    , TPriority pr                                                 
13                    , void (*exec)()                                               
14                #if scmRTOS_DEBUG_ENABLE == 1                                      
15                    , stack_item_t * aStackPool                                    
16                    , const char   * name = 0                                      
17                #endif                                                             
18                    );                                                             
19    protected:                                                                     
20        INLINE void set_unready() { Kernel.set_process_unready(this->Priority); }  
21        void init_stack_frame( stack_item_t * StackPoolEnd                         
22                             , void (*exec)()                                      
23        #if scmRTOS_DEBUG_ENABLE == 1                                              
24                             , stack_item_t * StackPool                            
25        #endif                                                                     
26                             );                                                    
27    public:                                                                        
28                                                                                   
29    #else  // SEPARATE_RETURN_STACK                                                
30                                                                                   
31        TBaseProcess( stack_item_t* StackPoolEnd                                   
32                    , stack_item_t* RStack                                         
33                    , TPriority pr                                                 
34                    , void (*exec)()                                               
35                #if scmRTOS_DEBUG_ENABLE == 1                                      
36                    , stack_item_t * aStackPool                                    
37                    , stack_item_t * aRStackPool                                   
38                    , const char   * name = 0                                      
39                #endif                                                             
40                    );                                                             
41    protected:                                                                     
42        void init_stack_frame( stack_item_t * Stack                                
43                             , stack_item_t * RStack                               
44                             , void (*exec)()                                      
45        #if scmRTOS_DEBUG_ENABLE == 1                                              
46                             , stack_item_t * StackPool                            
47                             , stack_item_t * RStackPool                           
48        #endif                                                                     
49                             );                                                    
50                                                                                   
51        TPriority   priority() const { return Priority; }                          
52                                                                                   
53        static void sleep(timeout_t timeout = 0);                                  
54               void wake_up();                                                     
55               void force_wake_up();                                               
56        INLINE void start() { force_wake_up(); }                                   
57                                                                                   
58        INLINE bool is_sleeping() const;                                           
59        INLINE bool is_suspended() const;                                          
60                                                                                   
61    #if scmRTOS_DEBUG_ENABLE == 1                                                  
62      INLINE TService * waiting_for() const { return WaitingFor; }                 
63    public:                                                                        
64               size_t       stack_size()  const { return StackSize; }              
65               size_t       stack_slack() const;                                   
66               const char * name()        const { return Name; }                   
67    #endif // scmRTOS_DEBUG_ENABLE                                                 
68                                                                                   
69    #if scmRTOS_PROCESS_RESTART_ENABLE == 1                                        
70    protected:                                                                     
71               void reset_controls();                                              
72    #endif                                                                         
73                                                                                   
74        //-----------------------------------------------------                    
75        //                                                                         
76        //    Data members                                                         
77        //                                                                         
78    protected:                                                                     
79        stack_item_t *     StackPointer;                                           
80        volatile timeout_t Timeout;                                                
81        const TPriority    Priority;                                               
82    #if scmRTOS_DEBUG_ENABLE == 1                                                  
83        TService           * volatile WaitingFor;                                  
84        const stack_item_t * const    StackPool;                                   
85        const size_t                  StackSize; // as number of stack_item_t items
86        const char                  * Name;                                        
87    #endif // scmRTOS_DEBUG_ENABLE                                                 
88                                                                                   
89    #if scmRTOS_PROCESS_RESTART_ENABLE == 1                                        
90        volatile TProcessMap * WaitingProcessMap;                                  
91    #endif                                                                         
92                                                                                   
93    #if scmRTOS_SUSPENDED_PROCESS_ENABLE != 0                                      
94        static TProcessMap SuspendedProcessMap;                                    
95    #endif                                                                         
96    };                                                                             
```

/// Caption  
Listing 1. TBaseProcess  
///

Despite the seemingly extensive class definition, `TBaseProcess` is actually quite small and simple. Its data representation consists of just three core members: the stack pointer (line 79), the timeout tick counter (line 80), and the priority value (line 81). The remaining data members are auxiliary and appear only when additional features are enabled&nbsp;– such as the ability to interrupt and restart a process at any point, or debugging support[^3].

[^3]: This applies to the rest of the code as well—the majority of the class definition is devoted to these optional capabilities.

The class interface provides the following functions:

* `sleep(timeout_t timeout = 0)`. Puts the process into a sleeping state: the argument value is assigned to the internal timeout counter, the process is removed from the ready-to-run process map, and the scheduler is invoked to transfer control to the next ready process.
* `wake_up()`. Wakes the process from sleep. The process is marked ready only if it was waiting for an event with a timeout; if its priority is higher than the current process, it immediately receives control.
* `force_wake_up()`. Forces the process out of sleep. The process is always marked ready. If its priority is higher than the current process, it immediately receives control. This function should be used with extreme caution, as incorrect usage can lead to unpredictable program behavior.
* `is_sleeping()`. Checks whether the process is sleeping (i.e., waiting for an event with a timeout).
* `is_suspended()`. Checks whether the process is in a suspended (inactive) state.

<a name="process-stack"></a>
### Stack

A process stack is a contiguous region of RAM used to store process data, save the process context, and hold return addresses from functions and interrupts.

Due to architectural features of some processors, two separate stacks may be used&nbsp;– one for data and one for return addresses. **scmRTOS** supports this capability, allowing each process object to contain two distinct RAM regions (two stacks), with sizes specified individually based on application requirements. Support for separate stacks is enabled via the `SEPARATE_RETURN_STACK` macro defined in `os_target.h`.

Within a protected section, a critically important function `init_stack_frame()` is declared, responsible for constructing the initial stack frame. The reason is that process executable functions do not start like ordinary functions&nbsp;– they are not called in the traditional way. Control reaches them through the same mechanism used for context switches between processes. Therefore, starting a process involves restoring its context from the stack followed by a jump to the address stored as the saved interrupt return point.

To enable this startup method, the process stack must be prepared accordingly: specific memory cells in the stack are initialized with required values, making the stack appear as if the process had previously been preempted (with its context properly saved). The exact steps for preparing the stack frame are platform-specific, so the implementation of `init_stack_frame()` is delegated to the OS port layer.

### Timeouts

Each process has a dedicated `Timeout` variable to control its behavior during event waits with timeouts or during sleep. Essentially, this variable acts as a down-counter of system timer ticks. When its value is non-zero, it is decremented in the system timer interrupt handler and tested against zero. Upon reaching zero, the owning process is marked ready to run.

Thus, if a process is put to sleep with a timeout (i.e., removed from the ready map via `sleep(timeout)` with a non-zero argument), it will be automatically awakened[^4] in the system timer interrupt handler after an interval corresponding to the specified number of system ticks[^5].

[^4]: I.e., marked ready-to-run.

[^5]: More precisely, the interval is accurate to within a fraction of one tick period, depending on the timing of the `sleep` call relative to the next timer interrupt.



The same mechanism applies when a service function is called that involves waiting for an event with a timeout. The process will be awakened either when the expected event occurs or when the timeout expires. The value returned by the service function unambiguously indicates the reason for awakening, allowing the user program to easily decide on subsequent actions.

### Priorities

Each process also has a data field holding its priority. This field serves as the process identifier when manipulating processes and their internal representation&nbsp;– in particular, the priority is used as an index into the kernel's process pointer table, where the address of each process is stored upon registration.

Priorities are unique&nbsp;– no two processes may share the same priority. The internal representation is an integer variable. For type safety when assigning priorities, a dedicated enumerated type `TPriority` is used.

<a name="process-sleep"></a>
### The sleep() Function

This function is used to transition the current process from an active state to an inactive one. If the function is called with an argument of 0 (or without specifying an argument&nbsp;– the function has a default argument of 0), the process will enter sleep indefinitely until it is explicitly awakened, for example, by another process using `TBaseProcess::force_wake_up()`. If called with a non-zero argument, the process will sleep for the specified number of system timer ticks, after which it will be automatically awakened (i.e., marked ready-to-run). In this case, the sleep can also be interrupted prematurely by another process or an interrupt handler using `TBaseProcess::wake_up()` or `TBaseProcess::force_wake_up()`.

----
## Creating and Using a Process

### Defining a Process Type

To create a process, its type must be defined and an object of that type declared.

A concrete process type is described using the `OS::process` template, see "Listing 2. Process Template".

```cpp
01    template<TPriority pr, size_t stk_size, TProcessStartState pss = pssRunning>     
02    class process : public TBaseProcess                                              
03    {                                                                                
04    public:                                                                          
05        INLINE_PROCESS_CTOR process( const char * name_str = 0, void (*func)() = 0 );
06                                                                                     
07        OS_PROCESS static void exec();                                               
08                                                                                     
09    #if scmRTOS_PROCESS_RESTART_ENABLE == 1                                          
10        INLINE void terminate( void (*func)() = 0 );                                 
11    #endif                                                                           
12                                                                                     
13    private:                                                                         
14        stack_item_t Stack[stk_size/sizeof(stack_item_t)];                           
15    };                                                                               
```

/// Caption  
Listing 2. Process Template  
///

As shown, two elements are added to what the base class provides:

* The process stack `Stack` with size `stack_size`. The size is specified in bytes.
* The static function `exec()`, which is the actual function containing the user code for the process.

### Declaring a Process Object and Using It

It is now sufficient to declare an object of this type—which becomes the process itself—and to define the process function `exec()`.

```cpp
typedef OS::process<OS::prN, 100> Slon;
Slon slon;
```

where `N` is the priority number.

["Listing 1. Process Executable Function in Overview section"](overview.md#process-exec) illustrates a typical example of a process function.

Using a process primarily involves writing user code inside the process function. As previously mentioned, a few simple rules must be followed:

* Care must be taken to ensure that program flow never exits the process function. Otherwise, since this function was not called in the conventional way, upon exit the flow of control would jump to undefined addresses, leading to undefined program behavior (though in practice, the behavior is usually quite defined&nbsp;– the program simply stops working!).
* The function `TBaseProcess::wake_up()` should be used cautiously and thoughtfully, while `TBaseProcess::force_wake_up()` requires particular care, as careless use can cause premature awakening of a sleeping (delayed) process, potentially leading to collisions in interprocess interaction.

<a name="process-alternate-exec"></a>
#### Alternative Ways to Declare a Process Object

##### External Function

When declaring a process object, a pointer to an external function of type `void func()` can be passed to the constructor; in this case, that function will serve as the process executable function, see "Listing 3. Using an external function as the executable".

```cpp
01    OS_PROCESS void slon_exec();          
02                                          
03    Slon slon("Slon Process", &slon_exec);
04                                          
05    void slon_exec()                      
06    {                                     
07        ... // Declarations               
08        ... // Init process’s data        
09        for(;;)                           
10        {                                 
11            ... // process’s main loop    
12        }                                 
13    }                                     
```
/// Caption
Listing 3. Using an external function as the executable
///

The advantage of this approach is a more concise notation without the need to specify full template specialization (`template<>`) and the namespace `OS`, which are required when using the member function `process::exec()`.

##### Executable Function as a Process Constructor Argument

In addition to a regular function, an anonymous function with the required signature can be passed to the process&nbsp;– this is implemented in C++ using lambda functions, see "Listing 4. Lambda Function as Process Executable Function".

```cpp
01    Slon slon("Slon Process", []      
02    {                                 
03        ... // Declarations           
04        ... // Init process’s data    
05        for(;;)                       
06        {                             
07            ... // process’s main loop
08        }                             
09    });                               
```
/// Caption
Listing 4. Lambda Function as Process Executable Function
///

The main advantage of this method is its conciseness: the process object and its executable function are contained in a single expression.

!!! warning "**NOTE**"

    Referring to "Listing 2. Process Template" (line 5), it can be seen that when an external function is used as the executable, the process name must also be specified&nbsp;– this is a requirement of C++ language syntax (default argument rules).
    
    In practice, the process name is used only for debugging purposes, so it is not mandatory, and the question may arise about additional overhead when the name is not needed. However, the process constructor implementation is such that no overhead occurs, see "Listing 5. Process Constructor".

    From the listing, it is evident that the process name is used only when debugging is enabled (line 04); otherwise, the `const char *` argument becomes unnamed and is removed from the code, so no overhead is introduced.

```cpp
01    template<TPriority pr, size_t stk_size, TProcessStartState pss>
02    OS::process<pr, stk_size, pss>::process( const char *          
03        #if scmRTOS_DEBUG_ENABLE == 1                              
04        name_str                                                   
05        #endif                                                     
06        , void (*func)()                                           
07        ) : TBaseProcess(&Stack[stk_size / sizeof(stack_item_t)]   
08                         , pr                                      
09                         , func ? func : exec                      
10                      #if scmRTOS_DEBUG_ENABLE == 1                
11                         , Stack                                   
12                         , name_str                                
13                      #endif                                       
14                         )                                         
15                                                                   
16    {                                                              
17        #if scmRTOS_SUSPENDED_PROCESS_ENABLE != 0                  
18        if ( pss == pssSuspended )                                 
19            clr_prio_tag(SuspendedProcessMap, get_prio_tag(pr));   
20        #endif                                                     
21    }                                                              
```
/// Caption
Listing 5. Process Constructor
///

### Starting a Process in a Suspended State

Sometimes it is necessary for a process's executable function to begin execution not immediately after system startup, but only upon receiving a specific signal. For example, several processes should start working only after some equipment (possibly external to the MCU) has been initialized/configured; otherwise, incorrect actions toward that equipment could have undesirable consequences.

In such cases, some form of dispatching is required&nbsp;– the processes must organize their operation in a way that preserves the correct interaction logic with the equipment. For instance, all processes except one (the dispatcher) could immediately wait at startup for a start event that will be signaled by the dispatcher process.

The dispatcher process performs all necessary preparatory work and then signals the start to the waiting processes. This approach requires manually adding appropriate waiting code to each process awaiting startup, which clutters the code, increases workload, and is error-prone.

There may also be other situations requiring delayed process activation. To support this functionality, a process can be configured to start in a so-called **suspended** state. Such a process is identical to any other except that its tag is absent from the ready-to-run process map (`ReadyProcessMap`).

Declaration of such a process looks like this[^6]:

```cpp
typedef OS::process<OS::pr1, 300, OS::pssSuspended> Proc2;
...
Proc2 proc2;
```

Later, to start this process, the initiating code must call the `force_wake_up()` function:

```cpp
Proc2.force_wake_up();
```

[^6]: The `ss` prefix in this example stands for **Start State**.

---
<a name="process-restart"></a>
## Process Restart

Situations may arise where it is necessary to externally interrupt a process and restart it from the beginning. For example, a process performs lengthy computations, but at some point the results become obsolete, and a new computation cycle with fresh data must be started. This can be achieved by terminating the current execution with the ability to restart the process from scratch.

To support this, the OS provides two functions to the user:

* `OS::process::terminate(void (*func)() = 0)`;
* `OS::TBaseProcess::start()`.

#### Terminate Process Execution

The `terminate()` function is intended to be called from outside the process being stopped. Inside it, all resources associated with the process are reset to their initial state, and the process is marked as not ready to run. If the process was waiting on a service, its tag is removed from that service's waiting process map.

The `terminate()` function can accept a pointer to a function as an argument; this function will serve as the executable entry point for the process on the next start. This provides considerable flexibility in program implementation&nbsp;– on each restart, the exact executable function required in the current program context can be specified.

!!! tip "**TIP**"
    The ability to specify the executable function on restart can be effectively used to simulate process deletion and creation&nbsp;– some libraries are designed to require dynamic resource allocation for their operation, in particular the creation of processes to perform tasks followed by their deletion.
    
    **scmRTOS** does not support dynamic process creation and deletion for reasons [described earlier](overview.md#avoid-dynamic-process), but creation/deletion can be simulated, for example by organizing a pool of processes from which an available process can be taken when needed and assigned an appropriate executable function.
    
    Changing process priorities or stack sizes is not possible—these parameters are set statically during OS configuration—but in many cases this is not required, since the resources needed to perform tasks are usually known at build time.

#### Start Process Execution

Starting the process is performed separately&nbsp;– allowing the user to do so at the moment they deem appropriate&nbsp;– using the `start()` function, which simply marks the process as ready-to-run. The process will resume execution according to its priority and the current OS load.

For process termination and restart to work correctly, this feature must be enabled during configuration—the macro `scmRTOS_PROCESS_RESTART_ENABLE` must be set to 1.
