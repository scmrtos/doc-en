# Kernel

---

## General Information

The OS kernel performs:

* Process organization functions;
* Scheduling at both process and interrupt levels;
* Support for **interprocess communication services**;
* System time support (system timer);
* Extension support.

The core of the system is the `TKernel` class, which includes all the necessary functions and data. For obvious reasons, there is only one instance of this class. Almost its entire implementation is private, and to allow access from certain OS parts that require kernel resources, the C++ "friend" mechanism is used — functions and classes granted such access are declared with the `friend` keyword.

It should be noted that in this context, the kernel refers not only to the `TKernel` object but also to the extension mechanism implemented as the `TKernelAgent` class. This class was specifically introduced to provide a base for building extensions. Looking ahead, all **interprocess communication services** in **scmRTOS** are implemented as such extensions. The `TKernelAgent` class is declared as a "friend" of `TKernel` and contains the minimal necessary set of protected functions to grant descendants access to kernel resources. Extensions are built by inheriting from `TKernelAgent`. For more details, see [TKernelAgent and Extensions](kernel.md#kernel-agent).

---

## TKernel. Composition and Operation

### Composition

The `TKernel` class contains the following data members[^1]:

* `CurProcPriority` — variable holding the priority number of the current active process. Used for quick access to the current process resources and for manipulating process status (both relative to the kernel and to interprocess communication services)[^2];
* `ReadyProcessMap` — map of processes ready for execution. Contains tags of ready processes: each bit corresponds to a specific process, with logical 1 indicating the process is ready[^3], and logical 0 indicating it is not;
* `ProcessTable` — array of pointers to processes registered in the system;
* `ISR_NestCount` — interrupt nesting counter variable. Incremented on each interrupt entry and decremented on each exit;
* `SysTickCount` — system timer tick (overflow) counter variable. Present only if this feature is enabled (via the corresponding macro in the configuration file);
* `SchedProcPriority`* — variable for storing the priority value of the process scheduled to receive control.

[^1]: Objects marked with ‘\*’ are present only in the variant using software interrupt-based control transfer.

[^2]: Ideologically, using a pointer to the process might seem more correct for these purposes, but analysis showed no performance gain, and the pointer size is typically larger than an integer variable for storing priority.

[^3]: The process may be active (executing) or inactive (waiting for control) — the latter occurs when there is another ready process with higher priority.

### Process Organization

The process organization function reduces to registering created processes. In each process constructor, the kernel function `register_process(TBaseProcess *)` is called, which places the pointer to the passed process into the system `ProcessTable` (see below). The position in the table is determined by the process priority, which effectively serves as the table index. The process registration function code is shown in "Listing 1. Process Registration Function".

```cpp
void OS::TKernel::register_process(OS::TBaseProcess * const p)
{
    ProcessTable[p->Priority] = p;
}
```

/// Caption
Listing 1. Process Registration Function
///

The next system function is the actual OS startup. The system startup function code is shown in "Listing 2. OS Startup Function".

```cpp
INLINE void OS::run()
{
    stack_item_t *sp = Kernel.ProcessTable[pr0]->StackPointer;
    os_start(sp);
}
```

/// Caption
Listing 2. OS Startup Function
///

As seen, the actions are extremely simple: the stack pointer of the highest-priority process is retrieved from the process table (line 3), and the system is started (line 4) by calling the low-level `os_start()` function, passing it the retrieved stack pointer of the highest-priority process.

From this moment, the OS begins operating in normal mode — control is transferred between processes according to their priorities, events, and the user program.

### Control Transfer

Control transfer can occur in two ways:

* The process voluntarily yields control when it has nothing more to do (for now), or as a result of its work, it needs to engage in interprocess communication with other processes (acquire a mutual exclusion semaphore (`OS::TMutex`), or, after signaling an event flag (`OS::TEventFlag`), notify the kernel, which must then perform (if necessary) process rescheduling;
* Control is taken from the process by the kernel due to an interrupt triggered by some event; if a higher-priority process was waiting for that event, control is given to it, and the interrupted process waits until the higher-priority one completes its task and yields control[^4].

[^4]: This higher-priority process may in turn be interrupted by an even higher-priority one, and so on, until the highest-priority process is reached — it can only be (temporarily) interrupted by an interrupt handler, but upon return, control always goes back to it. Thus, the highest-priority process cannot be preempted by any other process. Upon exiting an interrupt handler, control always passes to the highest-priority ready process.

In the first case, rescheduling is synchronous relative to program execution flow — in the scheduler code. In the second case, it is asynchronous upon event occurrence.

Control transfer itself can be organized in several ways. One is direct transfer by calling a low-level[^6] context switcher function from the scheduler[^5]. Another is by triggering a special software interrupt where the context switch occurs. **scmRTOS** supports both methods. Each has advantages and disadvantages, discussed in detail below.

[^5]: Or upon interrupt handler exit — depending on whether the transfer is synchronous or asynchronous.

[^6]: Usually implemented in assembly.

### Scheduler

The scheduler source code is in the `sched()` function — see "Listing 3. Scheduler".

There are two variants: one for direct control transfer (`scmRTOS_CONTEXT_SWITCH_SCHEME == 0`), the other for software interrupt-based transfer.

Note that scheduling from the main program level is done via the `scheduler()` function, which calls the actual scheduler only if not invoked from an interrupt:

```cpp
INLINE void scheduler() { if(ISR_NestCount) return; else sched(); }
```

With proper use of OS services, this situation should not occur, as scheduling from interrupts should use specialized versions of functions (names suffixed with `_isr`) designed for interrupt level.

For example, to signal an event flag from an interrupt, the user should use `signal_isr()`[^7] instead. However, using the non-\_isr version won't cause a fatal error — the scheduler simply won't be called, and despite the event arriving in the interrupt, no control transfer occurs, even if it was due.

Control transfer happens only at the next rescheduling call, which occurs when the destructor of a `TISRW/TISRW_SS` object executes. Thus, `scheduler()` provides protection against program crashes from careless service use or services lacking `_isr` versions — e.g., `channel::push()`.

[^7]: All interrupt handlers using interprocess communication services must declare a `TISRW` object before any service call (i.e., where scheduling may occur). This object must be declared before the first OS service use.

```cpp
01    bool OS::TKernel::update_sched_prio()
02    {
03        uint_fast8_t NextPrty = highest_priority(ReadyProcessMap);
04    
05        if(NextPrty != CurProcPriority)
06        {
07            SchedProcPriority = NextPrty;
08            return true;
09        }
10    
11        return false;
12    }
    
13    #if scmRTOS_CONTEXT_SWITCH_SCHEME == 0
14    void TKernel::sched()
15    {
16        uint_fast8_t NextPrty = highest_priority(ReadyProcessMap);
17        if(NextPrty != CurProcPriority)
18        {
19        #if scmRTOS_CONTEXT_SWITCH_USER_HOOK_ENABLE == 1
20            context_switch_user_hook();
21        #endif
22    
23            stack_item_t*  Next_SP      = ProcessTable[NextPrty]->StackPointer;
24            stack_item_t** Curr_SP_addr = &(ProcessTable[CurProcPriority]->StackPointer);
25            CurProcPriority = NextPrty;
26            os_context_switcher(Curr_SP_addr, Next_SP);
27        }
28    }
29    #else
30    void TKernel::sched()
31    {
32        if(update_sched_prio())
33        {
34            raise_context_switch();
35            do
36            {
37                enable_context_switch();
38                DUMMY_INSTR();
39                disable_context_switch();
40            }
41            while(CurProcPriority != SchedProcPriority); // until context switch done
42        }
43    }
44    #endif // scmRTOS_CONTEXT_SWITCH_SCHEME
```

/// Caption
Listing 3. Scheduler
///

#### Scheduler with Direct Control Transfer

All actions inside the scheduler must be non-interruptible, so the function code executes in a critical section. However, since the scheduler is always called with interrupts disabled, no explicit critical section is needed.

First, the priority of the highest-priority ready process is computed (by analyzing the ready process map `ReadyProcessMap`).

The found priority is compared to the current one. If they match, the current process is the highest-priority ready one, no transfer is needed, and execution continues in the current process.

If they differ, a higher-priority ready process has appeared, and control must transfer to it via context switch. The current process context is saved to its stack, and the next process context is restored from its stack. These platform-dependent actions are performed in the low-level (assembly) `os_context_switcher()` function called from the scheduler (line 26). It receives two parameters:

* Address of the current process stack pointer, where the pointer itself will be stored after saving the current context (line 24);
* Stack pointer of the next process (line 23).

When implementing the low-level context switcher, pay attention to the platform and compiler calling conventions and parameter passing.

#### Scheduler with Software Interrupt

This variant differs significantly from the above. The main difference is that the actual context switch occurs not by directly calling the context switcher but by triggering a special software interrupt where the switch happens. This approach has nuances and requires special measures to prevent system integrity violations.

The primary challenge in implementing this control transfer method is that the scheduler code and the software interrupt handler code are not strictly continuous or "atomic"—an interrupt can occur between them, potentially triggering another rescheduling and causing an overlap that corrupts the control transfer process. To avoid this collision, the "rescheduling-control transfer" process is divided into two "atomic" operations that can be safely separated.

The first operation is, as before, computing the priority of the highest-priority ready process—via the call to `update_sched_prio()` (line 01)—and checking whether rescheduling is necessary (line 32). If it is, the priority value of the next process is stored in the `SchedProcPriority` variable (line 07), and the software context switch interrupt is raised (line 34). The program then enters a loop waiting for the context switch to occur (line 35).

This hides a rather subtle point. Why not, for example, simply implement the interrupt-enabled window with a pair of dummy instructions (to give the processor hardware time to actually trigger the interrupt)? Such an implementation conceals a hard-to-detect error, as follows.

If, at the moment interrupts are enabled—which in this OS version is implemented by globally enabling interrupts (line 37)—one or more other interrupts are pending in addition to the software interrupt, and some of them have higher priority than the software context switch interrupt, control will naturally transfer to the handler of the corresponding interrupt. Upon completion, execution returns to the interrupted program. At this point, in the main program (i.e., inside the scheduler function), the processor may execute one or more instructions[^8] before the next interrupt can be serviced.

[^8]: This is a common property of many processors—after returning from an interrupt, transitioning to the next interrupt handler is not possible immediately in the same machine cycle but only after one or more cycles.

The program could then reach the code that disables context switching, resulting in interrupts being globally disabled and preventing the software interrupt (where the context switch occurs) from executing. This means control would remain in the current process, even though it should have been transferred to the system (and other processes) until the event awaited by the current process occurs. This is nothing less than a violation of system integrity and can lead to a wide variety of unpredictable negative consequences.

Clearly, such a situation must not arise. Therefore, instead of a few dummy instructions in the interrupt-enabled window, a context switch wait loop is used. No matter how many interrupts are queued, program control does not proceed beyond this loop until the actual context switch has occurred.

To make this mechanism work, a criterion is needed to confirm that rescheduling has actually taken place. This criterion is the equality of the kernel variables `CurProcPriority` and `SchedProcPriority`. These variables become equal (i.e., the current priority value matches the scheduled one) only after the context switch has been performed.

As can be seen, no updates are made here to variables holding stack pointers or the current priority value. All such actions are performed later during the actual context switch by calling the special kernel function `os_context_switch_hook()`.

One might ask: why all this complexity? To answer, consider a scenario where, in the software interrupt case, the scheduler implementation remained the same as in the direct context switcher call—only instead of:

```cpp
os_context_switcher(Curr_SP_addr, Next_SP);
```

we have[^9]:

```cpp
raise_context_switch();
<wait_for_context_switch_done>;
```

[^9]: Here, `<wait_for_context_switch_done>` represents all the code ensuring the context switch, starting from enabling interrupts.

Now imagine a situation where, at the moment interrupts are enabled, one or more other interrupts are pending, at least one of which is higher priority than the software context switch interrupt, and the handler for that higher-priority pending interrupt calls one of the interprocess communication service functions. What happens then?

The scheduler would be invoked again, triggering another process rescheduling. However, since the previous rescheduling was not completed—i.e., processes were not actually switched, contexts were not physically saved and restored—the new rescheduling would simply overwrite the variables holding the current and next process pointers.

Moreover, when determining the need for rescheduling, the value of `CurProcPriority` would be used, which is effectively incorrect because it holds the priority of the process scheduled from the previous scheduler invocation. In short, rescheduling operations would overlap, violating system integrity.

Therefore, it is critical that the actual update of `CurProcPriority` and the process context switch be "atomic"—inseparable and not interrupted by other code related to process scheduling. In the direct context switcher call variant, this rule is inherently satisfied: the entire scheduler operates in a critical section, and the context switcher is called directly from there.

In the variant with software interrupt, context scheduling and switching can be "separated" in time. Therefore, the actual switching and updating of the current priority occur directly during the execution of the software interrupt handler[^10]. In it, immediately after saving the context of the current process, the function `os_context_switch_hook()` is called (where the value of `CurProcPriority` is actually updated), and the stack pointer of the current process is passed to `os_context_switch_hook()`, where it is saved in the current process object. The stack pointer of the next process is then retrieved and returned from the function, which is necessary for restoring the context of that process and subsequently transferring control to it.

To avoid degrading performance characteristics in interrupt handlers, there is a special lightweight embedded version of the scheduler used by some member functions of service objects, optimized for use in ISRs. The code for this scheduler version is shown in "Listing 4. Scheduler variant optimized for use in ISR".

[^10]: This software interrupt handler is always implemented in assembly and is also platform-dependent, so its code is not provided here.

```cpp
01    void OS::TKernel::sched_isr()
02    {
03        if(update_sched_prio())
04        {
05            raise_context_switch();
06        }
07    }
```

/// Caption  
Listing 4. Scheduler variant optimized for use in ISR  
///

When selecting an interrupt handler for context switching, preference should be given to one with the lowest priority (in the case of a priority interrupt controller). This avoids unnecessary rescheduling and context switches if multiple interrupts occur in succession.

### Pros and Cons of Control Transfer Methods  
Both methods have their advantages and disadvantages. The strengths of one control transfer method are the weaknesses of the other, and vice versa.

#### Direct Control Transfer  
The main advantage of direct control transfer is that it does not require a special software interrupt in the target MCU—not all MCUs have this hardware capability. A secondary minor benefit is slightly higher performance compared to the software interrupt variant, as the latter incurs additional overhead for activating the context switch interrupt handler, the wait cycle for context switching, and the call to `os_context_switch_hook()`.

However, the direct control transfer variant has a significant drawback: when the scheduler is called from an interrupt handler, the compiler is forced to save the "local context" (scratch registers of the processor) due to the call to a non-inlined context switch function, which introduces overhead that can be substantial compared to the rest of the ISR code. The negative aspect here is that saving these registers may be entirely unnecessary—after all, in that function[^11], which causes them to be saved, these registers are not used. Therefore, if there are no further calls to non-inlined functions, the code for saving and restoring this group of registers turns out to be redundant.

[^11]:  
```cpp
os_context_switcher(stack_item_t **Curr_SP, stack_item_t *Next_SP)
```

#### Software Interrupt-Based Control Transfer  
This variant avoids the aforementioned drawback. Since the ISR itself executes normally without rescheduling from within it, saving the "local context" is also not performed, significantly reducing overhead and improving system performance. To avoid spoiling the picture by calling a non-inlined member function of an interprocess communication service object, it is recommended to use special lightweight, inlinable versions of such functions—for more details, see [the Inter-Process Communication section]([ipcs.md](http://ipcs.md)).

The main disadvantage of software interrupt-based control transfer is that not all hardware platforms support software interrupts. In such cases, one of the unused hardware interrupts can be used as a software interrupt. Unfortunately, this introduces some lack of universality—it is not known in advance whether a particular hardware interrupt will be needed in a given project. Therefore, if the processor does not specifically provide a suitable interrupt, the choice of context switch interrupt is delegated (from the port level) to the project level, and the user must write the corresponding code[^12] themselves.

[^12]: The **scmRTOS** distribution is offered with several working usage examples, where all the code for organizing and configuring the software interrupt is present. Thus, the user can simply modify this code to suit their project's needs or use it as-is if everything fits.

When using software interrupt-based control transfer, the expression "The kernel takes control away from processes" fully reflects the situation.

#### Conclusions  
Given the above analysis of the advantages and disadvantages of both control transfer methods, the general recommendation is as follows: if the target platform provides a suitable interrupt for implementing context switching, it makes sense to use this variant, especially if the size of the "local context" is sufficiently large.

Using direct control transfer is justified when it is truly impossible to use a software interrupt—for example, when the target platform does not support such an interrupt, and using a hardware interrupt as a software one is impossible for one reason or another, or if the performance characteristics with this control transfer variant prove better due to lower overhead in organizing context switches, while saving/restoring the "local context" does not introduce noticeable overhead due to its small size[^13].

[^13]: For example, on **MSP430**/IAR, the "local context" consists of just 4 registers.

### Support for Interprocess Communication  
Support for interprocess communication boils down to providing a set of functions for monitoring process states, as well as granting access to rescheduling mechanisms for the OS components—interprocess communication facilities. For more details on this, see [the Inter-Process Communication section](ipcs.md).

### Interrupts

#### Usage Features with RTOS and Implementation

An occurring interrupt can serve as a source of an event that requires handling by one or more processes. To minimize (and ensure determinism of) the response time to the event, process rescheduling is used when necessary, transferring control to the highest-priority process that is ready to run.

The code of any interrupt handler that uses interprocess communication services must call the function `isr_enter()` at the beginning, which increments the variable `ISR_NestCount`, and call the function `isr_exit()` at the end, which decrements `ISR_NestCount` and determines the interrupt nesting level (in the case of nested interrupts) based on its value. When `ISR_NestCount` reaches zero, it indicates a return from the interrupt handler to the main program, and `isr_exit()` performs process rescheduling (if required) by invoking the interrupt-level scheduler.

To simplify usage and improve portability, the code executed at the entry and exit of interrupt handlers is placed in the constructor and destructor, respectively, of a special wrapper class—`TISRW`. An object of this type must be used within the interrupt handler[^14]. It is sufficient to create an object of this type in the interrupt handler code; the compiler will handle the rest automatically. Importantly, the declaration of this object must precede the first use of any service functions.

[^14]: The aforementioned functions `isr_enter()` and `isr_exit()` are member functions of this wrapper class.

It should be noted that if a non-inlinable function is called within an interrupt handler, the compiler will save the "local context"—the scratch[^15] registers[^16]. Therefore, it is advisable to avoid calls to non-inlinable functions from interrupt handlers, as even partial context saving degrades both execution speed and code size[^17]. For this reason, in the current version of **scmRTOS**, some interprocess communication objects have been augmented with special lightweight functions designed for use in interrupt handlers. These functions are inlinable and employ a lightweight version of the scheduler, which is also inlinable. For more details, see [the Interprocess Communication Services section](ipcs.md).

[^15]: Typically, the compiler divides processor registers into two groups: scratch and preserved. Scratch registers are those that any function may use without prior saving. Preserved registers are those whose values must be saved if the function needs to use them (the function must save the value before use and restore it afterward). In some cases, preserved registers are referred to as local; in the context discussed here, these terms are synonymous.

[^16]: The proportion of these registers (relative to the total number) varies across platforms. For example, when using EWAVR, they account for roughly half of all registers; with EW430, less than half. In the case of VisualDSP++/**Blackfin**, the proportion of these registers is large, but on this platform, stack sizes are generally large enough that this is not a major concern.

[^17]: Unfortunately, when using the direct control transfer scheme, a non-inlinable context switch function is called, so the overhead of saving scratch registers cannot be avoided in this case.

#### Separate Interrupt Stack and Nested Interrupts

Another aspect related to interrupts in a preemptive RTOS is the use of a separate stack for interrupt handlers. As is well known, when an interrupt occurs and control is transferred to its handler, the program uses the stack of the interrupted process. This stack must be large enough to satisfy the needs of both the process itself and any interrupt handler. Moreover, it must accommodate the combined worst-case requirements—for example, when the process code has reached its peak stack usage and an interrupt occurs at that moment, with its handler also consuming additional stack space. The stack size must be sufficient to prevent overflow even in this scenario.

Clearly, the above considerations apply to all processes in the system. If interrupt handlers consume a significant amount of stack space, the stack sizes of all processes must be increased by a corresponding amount. This leads to higher memory overhead. In the case of nested interrupts, the situation becomes dramatically worse.

To mitigate this effect, the processor's stack pointer is switched to a dedicated interrupt stack upon entry into an interrupt handler. This effectively decouples the process stacks from the interrupt stack, eliminating the need to reserve additional memory in each process stack for interrupt handler operation.

The implementation of a separate interrupt stack is handled at the port level. Some processors provide hardware support for switching the stack pointer to the interrupt stack, making this feature efficient and safe[^18].

[^18]: In such cases, this mechanism is the only one implemented in the port, and there is no need for a separate implementation of the `TISRW_SS` wrapper class.

Nested interrupts—those whose handlers can interrupt not only the main program but also other interrupt handlers—have specific usage characteristics. Understanding these is essential for effective and safe application of the mechanism. When the processor has a priority-based interrupt controller supporting multiple priority levels, handling nested interrupts is relatively straightforward. Potential dangerous situations when enabling nesting are typically accounted for by the processor designers, and the interrupt controller prevents issues such as those described below.

In processors with a single-level interrupt system, the typical implementation automatically disables interrupts globally upon any interrupt occurrence—for reasons of simplicity and safety. In other words, nested interrupts are not supported. To enable nesting, it is sufficient to globally re-enable interrupts, which are usually disabled by hardware when control is transferred to the handler. However, this can lead to a situation where an already executing interrupt handler is invoked again—if the interrupt request for the same source remains pending[^19].

[^19]: This may occur, for example, due to events triggering the interrupt too frequently or because the interrupt flag was not cleared, continuing to assert the request.

This is generally an erroneous situation that must be avoided. To prevent it, one must clearly understand both the processor's operational specifics and its current "context"[^20], and write code very carefully: before globally enabling interrupts, disable the activation of the interrupt whose handler is already running (to avoid re-entry into the same handler), and upon completion, remember to restore the processor's control resources to their original state before the nesting manipulations.

Based on the above, the following recommendation can be made.

[^20]: Here, "context" refers to the logical and semantic environment in which this part of the program executes.

!!! error "**WARNING**"
    Despite the apparent advantages of a separate interrupt stack, it is not recommended on processors lacking hardware support for switching the stack pointer to the interrupt stack.
    This is due to additional overhead from manual stack switching, poor portability—any non-standard extensions are a source of problems—and the fact that direct manipulation of the stack pointer can cause collisions with local object addressing. For example, the compiler, seeing the body of the interrupt handler, allocates[^21] memory for local objects on the stack—and does so before calling[^22] the wrapper constructor. As a result, after switching the stack pointer to the interrupt stack, the previously allocated memory will physically reside elsewhere, causing the program to malfunction, while the compiler cannot detect this issue.
    Similarly, nested interrupts are not recommended on processors without hardware support for them. Such interrupts require careful handling and usually additional maintenance—for example, blocking the interrupt source to prevent re-invocation of the same handler when interrupts are enabled.

[^21]: More precisely—reserves. This is typically done by modifying the stack pointer.
[^22]: And it has every right to do so.

Brief conclusion: The motivation for using a separate interrupt stack correlates with the use of nested interrupts—since nesting significantly increases stack consumption in interrupt handlers, imposing—in the absence of a separate interrupt stack—additional requirements on process stack sizes[^23].

!!! tip "**TIP**"
    When using a preemptive RTOS, it is possible to structure the program so that interrupt handlers serve only as event sources, with all event processing moved to the process level. This keeps interrupt handlers small and fast, in turn eliminating the need for both a separate interrupt stack and nested interrupt support. In this case, the interrupt handler body can be comparable in size to the overhead of switching to a separate interrupt stack and enabling nesting.

[^23]: Moreover, each process must have a stack large enough to cover both its own needs and the stack consumption of interrupt handlers, including the full nesting hierarchy.

This approach is precisely what is recommended when the processor lacks hardware support for switching to a separate interrupt stack and does not have an interrupt controller with hardware nested interrupt support.

It should be noted that a priority-based preemptive RTOS is, in a sense, analogous to a multi-level priority interrupt controller—it provides the ability to distribute code execution according to importance/urgency. For this reason, in most cases there is no need to place event-processing code at the interrupt level even when such a hardware controller is present; instead, use interrupts solely as event sources[^24] and move their processing to the process level. This is the recommended programming style.

[^24]: Making interrupt handlers as simple, short, and fast as possible.

### System Timer

The system timer is used to generate specific time intervals required for process operation, including support for timeouts.

Typically, one of the processor's hardware timers is used as the system timer[^25].

The system timer functionality is implemented in the kernel function `system_timer()`. The code for this function is shown in "Listing 5. System Timer".

[^25]: The simplest timer (without advanced features) is suitable for this purpose. The only fundamental requirement is that it must be capable of generating periodic interrupts at equal intervals—for example, an overflow interrupt. It is also desirable to have the ability to control the overflow period in order to select an appropriate system tick frequency.

```cpp
01 void OS::TKernel::system_timer()
02 {
03     SYS_TIMER_CRIT_SECT();
04 #if scmRTOS_SYSTEM_TICKS_ENABLE == 1
05     SysTickCount++;
06 #endif
07
08 #if scmRTOS_PRIORITY_ORDER == 0
09     const uint_fast8_t BaseIndex = 0;
10 #else
11     const uint_fast8_t BaseIndex = 1;
12 #endif
13
14     for(uint_fast8_t i = BaseIndex; i < (PROCESS_COUNT-1 + BaseIndex); i++)
15     {
16         TBaseProcess *p = ProcessTable[i];
17
18         if(p->Timeout > 0)
19         {
20             if(--p->Timeout == 0)
21             {
22                 set_process_ready(p->Priority);
23             }
24         }
25     }
26 }
```

/// Caption  
Listing 5. System Timer  
///

As can be seen from the source code, the actions are very straightforward:

1. If the tick counter is enabled, the tick counter variable is incremented (line 5);
2. Then, in a loop, the timeout values of all registered processes are checked. If the checked value is not zero[^26], it is decremented and tested for zero. When it reaches zero (after decrement), meaning the process timeout has expired, the process is marked as ready to run.

[^26]: This indicates that the process is waiting with a timeout.

Since this function is called inside the timer interrupt handler, upon returning to the main program (as described earlier), control will be transferred to the highest-priority ready process. Thus, if the timeout of a process with higher priority than the interrupted one has expired, that process will receive control after exiting the interrupt. This is achieved through the scheduler (see above)[^27].

[^27]: Provided it is possible to set the priority of the system timer interrupt.

!!! info "**NOTE**"
    Some RTOSes provide recommendations for the system tick duration, most commonly suggesting a range of 10[^28]–100 ms. This may be appropriate for those systems. The trade-off here is between minimizing overhead from system timer interrupts and achieving finer time resolution.
    
    Given that **scmRTOS** targets small microcontrollers operating in real-time environments, and considering that execution overhead[^29] is very low, the recommended system tick period is 1–10 ms.
    
    An analogy can be drawn with other domains where smaller objects typically operate at higher frequencies: for example, a mouse's heartbeat is much faster than a human's, and a human's is faster than an elephant's, with agility being inversely related. A similar trend exists in engineering, so it is reasonable to expect shorter tick periods on smaller processors than on larger ones—in larger systems, overhead is generally higher due to greater loading of the more powerful processor and, consequently, reduced responsiveness.

[^28]: For example, how would one implement dynamic LED display multiplexing with such a period when it is known that, for comfortable viewing with four digits, the digit refresh period must not exceed 5 ms?

[^29]: Due to the small number of processes and the simple, fast scheduler.

----

<a name="kernel-agent"></a>
## TKernelAgent and Extensions

### Kernel Agent

The `TKernelAgent` class is a specialized mechanism designed to provide controlled access to kernel resources when developing extensions to the operating system's functionality.

The overall concept is as follows: creating any functional extension for the OS requires access to certain kernel resources—such as the variable holding the priority of the active process or the system process map. Granting direct access to these internal structures would be unwise, as it violates the security model of object-oriented design[^30] (specifically, the principles of encapsulation and abstraction). This could lead to negative consequences, such as program instability due to insufficient coding discipline or loss of compatibility if the internal kernel representation changes.

To address this, an approach based on a dedicated class—the **kernel agent**—is proposed. It restricts access through a documented interface, allowing extensions to be created in a formalized, simpler, and safer manner.

[^30]: Principles of encapsulation and abstraction.

The code for the kernel agent class is shown in "Listing 6. TKernelAgent".

```cpp
01 class TKernelAgent
02 {
03     INLINE static TBaseProcess * cur_proc() { return Kernel.ProcessTable[cur_proc_priority()]; }
04
05 protected:
06     TKernelAgent() { }
07     INLINE static uint_fast8_t const & cur_proc_priority() { return Kernel.CurProcPriority; }
08     INLINE static volatile TProcessMap & ready_process_map() { return Kernel.ReadyProcessMap; }
09     INLINE static volatile timeout_t & cur_proc_timeout() { return cur_proc()->Timeout; }
10     INLINE static void reschedule() { Kernel.scheduler(); }
11
12     INLINE static void set_process_ready (const uint_fast8_t pr) { Kernel.set_process_ready(pr); }
13     INLINE static void set_process_unready (const uint_fast8_t pr) { Kernel.set_process_unready(pr); }
14
15 #if scmRTOS_DEBUG_ENABLE == 1
16     INLINE static TService * volatile & cur_proc_waiting_for() { return cur_proc()->WaitingFor; }
17 #endif
18
19 #if scmRTOS_PROCESS_RESTART_ENABLE == 1
20     INLINE static volatile
21     TProcessMap * & cur_proc_waiting_map() { return cur_proc()->WaitingProcessMap; }
22 #endif
23 };
```

/// Caption  
Listing 6. TKernelAgent  
///

As can be seen from the code, the class is defined in such a way that instances of it cannot be created. This is intentional: `TKernelAgent` is designed to serve as a base for building extensions. Its primary role is to provide a documented interface to kernel resources. Therefore, its functionality becomes available only through derived classes, which represent the actual extensions.

An example of using `TKernelAgent` will be discussed in more detail below when describing the base class for interprocess communication services—`TService`.

The entire interface consists of inline functions, which in most cases allows extensions to be implemented without sacrificing performance compared to direct access to kernel resources.

### Extensions

The kernel agent class described above enables the creation of additional features that extend the OS capabilities. The methodology for creating such extensions is straightforward: simply declare a class derived from `TKernelAgent` and define its contents. Such classes are referred to as **operating system extensions**.

The layout of the OS kernel code is organized so that class declarations and definitions of certain class member functions are separated into the header file `os_kernel.h`. This allows a user-defined class to have access to all kernel type definitions while simultaneously making the user-defined class visible to member functions of kernel classes—for example, in the scheduler and the system timer function[^31].

[^31]: In user hooks.

Extensions are integrated using the configuration file `scmRTOS_extensions.h`, which is included in `os_kernel.h` between the kernel type definitions and their member function implementations. This makes it possible to place the extension class definition in a separate user header file and include it in the project by adding it to `scmRTOS_extensions.h`. Once done, the extension is ready for use according to its intended purpose.
