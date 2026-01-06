# OS Kernel

----

## General Information

The operating system kernel performs the following functions:

*   Process organization and management.
*   Scheduling at both the process and interrupt levels.
*   Support for inter-process communication.
*   System time support (system timer).
*   Support for extensions.

The core of the system is the `TKernel` class, which contains all the necessary set of functions and data. For obvious reasons, only a single instance of this class exists. Almost all of its internal representation is private. To allow access from certain parts of the OS that require its resources, the C++ "friend" mechanism is used — functions and classes granted such access are declared with the `friend` keyword.

It is important to note that in this context, the term "kernel" refers not only to the `TKernel` object but also to the functional extension facility implemented as the `TKernelAgent` class. This class was specifically introduced into the operating system to provide a base for building extensions. Looking ahead, it can be noted that in **scmRTOS**, all inter-process communication facilities are implemented based on such an extension. The `TKernelAgent` class is declared as a `friend` of the `TKernel` class and contains a minimal necessary set of `protected` functions to provide its descendants with access to kernel resources. Extensions are built by inheriting from the `TKernelAgent` class. For more details, see [TKernelAgent and Extensions](kernel.md#kernel-agent).

----

## TKernel. Composition and Operation

### Composition

The `TKernel` class contains the following data members[^1]:

*   `CurProcPriority`&nbsp;– A variable containing the priority number of the currently active process. Serves for quick access to the current process's resources and for manipulating the process's status (both in relation to the kernel and to IPC facilities)[^2].
*   `ReadyProcessMap`&nbsp;– A map of processes ready for execution. Contains tags of processes ready to run: each bit of this variable corresponds to a specific process. A logical `1` indicates that the process is ready for execution[^3], a logical `0` indicates that it is not ready.
*   `ProcessTable`&nbsp;– An array of pointers to processes registered in the system.
*   `ISR_NestCount`&nbsp;– A counter for interrupt entries. It is incremented on each entry and decremented on each exit.
*   `SysTickCount`&nbsp;– A counter for system timer ticks (overflows). Present only if this feature is enabled (via the corresponding macro in the configuration file).
*   `SchedProcPriority`*&nbsp;– A variable for storing the priority value of the process scheduled to receive control.

[^1]: Objects marked with ‘\*’ are present only in the variant using software interrupt-based control transfer.
[^2]: Perhaps, from an ideological standpoint, using a pointer to the process for these purposes would be more correct. However, analysis showed no performance gain, and the size of a pointer is typically larger than that of an integer variable for storing a priority.
[^3]: At the same time, the process can be active (i.e., executing) or inactive (i.e., waiting to gain control). The latter situation occurs when there is another ready-to-run process in the system with a higher priority.

### Process Setup

The process setup essentially involves registering created processes. In the constructor of each process, the kernel function `register_process(TBaseProcess *)` is called. This function places the pointer to the process (passed as an argument) into the system's `ProcessTable` (see below). The position of this pointer in the table is determined according to the process's priority, which effectively serves as an index for accessing the table. See "Listing 1. Process Registration Function" for the code.

```cpp
1    void OS::TKernel::register_process(OS::TBaseProcess * const p)
2    {
3        ProcessTable[p->Priority] = p;
4    }
```
/// Caption
Listing 1. Process Registration Function
///

The next system function is the actual OS startup. See "Listing 2. OS Startup Function" for the code.

```cpp
1    INLINE void OS::run()
2    {
3        stack_item_t *sp = Kernel.ProcessTable[pr0]->StackPointer;
4        os_start(sp);
5    }
```
/// Caption
Listing 2. OS Startup Function
///

The actions are quite simple: the stack pointer of the highest-priority process is retrieved from the process table (3), and the system is actually started (4) by calling the low-level function `os_start()` with the retrieved stack pointer as an argument.

From this moment, the OS begins operating in its primary mode, i.e., transferring control from process to process according to their priorities, events, and the user program.

### Control Transfer

Control can be transferred in two ways:

*   A process voluntarily relinquishes control when it has nothing more to do (for the moment), or as a result of its operation, it needs to engage in inter-process communication (e.g., acquire a mutex semaphore (`OS::TMutex`) or signal an event flag (`OS::TEventFlag`)). This informs the kernel, which must then perform process rescheduling if necessary.
*   Control is taken from a process by the kernel as a result of an interrupt triggered by some event. If a higher-priority process was waiting for this event, control will be given to that process. The interrupted process will wait until the higher-priority one completes its task and relinquishes control[^4].

[^4]: This higher-priority process can, in turn, be interrupted by an even higher-priority process, and so on, up to the highest-priority process. The highest-priority process can only be (temporarily) interrupted by a hardware interrupt, and upon return from that interrupt, control will always go back to this same highest-priority process. Thus, the highest-priority process cannot be preempted by any other process. Upon exiting an interrupt handler, control is always transferred to the highest-priority process ready for execution.

In the first case, process rescheduling is performed **synchronously** relative to the program execution flow — within the scheduler code. In the second case, rescheduling occurs **asynchronously** upon the occurrence of an event.

The actual control transfer can be organized in several ways. One method is **direct control transfer** by calling a low-level[^6] context switch function from the scheduler[^5]. Another method is control transfer by **activating a special software interrupt**, where the context switch takes place. **scmRTOS** supports both methods. Each approach has its own advantages and disadvantages, which will be discussed in detail later.

[^5]: Or upon exiting an interrupt handler — depending on whether the control transfer is synchronous or asynchronous.
[^6]: Usually implemented in assembly language.

### The Scheduler

The scheduler's source code is implemented in the `sched()` function — see "Listing 3. The Scheduler".

Two variants are present here — one for the case of direct control transfer (`scmRTOS_CONTEXT_SWITCH_SCHEME == 0`), and another for control transfer using a software interrupt.

It should be noted that invoking the scheduler from the main program level is done via the `scheduler()` function. This function calls the actual scheduler (`sched()`) only if the call is not made from within an interrupt:

```cpp
INLINE void scheduler() { if(ISR_NestCount) return; else  sched(); }
```

With proper use of the OS facilities, this situation should not occur, because invoking the scheduler from interrupt level should be done through specialized versions of the corresponding functions (their names have the `_isr` suffix), which are specifically designed for use within interrupts.

For example, if it is necessary to signal an event flag from within an interrupt, the user must use the `signal_isr()`[^7] function instead of `signal()`. However, if the latter is used by mistake, no fatal runtime error will occur; the scheduler simply will not be invoked. Consequently, despite an event possibly occurring within the interrupt, a control transfer will not happen, even if its turn has arrived.

The control transfer will only occur during the next call to reschedule, which happens when the destructor of the `TISRW`/`TISRW_SS` object is executed. Thus, the check in the `scheduler()` function serves as protection against program crashes due to careless use of services, as well as when using services for which corresponding `_isr` functions are not provided — for example, `channel::push()`.

[^7]: All interrupt handlers in a program that use inter-process communication facilities must contain a declaration of a `TISRW` object placed **before** any call to a service function (i.e., where the scheduler might be invoked). This object must be declared before the first use of any OS services.


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
Listing 3. The Scheduler
///

#### Scheduler with Direct Control Transfer

All actions performed inside the scheduler must be non-interruptible; therefore, the code of this function executes within a critical section. However, since the scheduler is always called with interrupts disabled, using a critical section in its code is unnecessary.

The first step is to calculate the priority of the highest-priority process ready for execution (by analyzing the `ReadyProcessMap`&nbsp;– the map of processes ready to run).

Next, the found priority is compared with the current one. If they match, the current process is indeed the highest-priority one ready to run, and no control transfer to another process is required; the execution flow remains within the current process.

If the found priority does not match the current one, it means a process with a higher priority than the current one has become ready to run, and control must be transferred to it. This is achieved by switching process contexts. The context of the current process is saved onto its stack, and the context of the next process is restored from its stack. These actions are platform-dependent and are performed in the low-level (assembly-implemented) function `os_context_switcher()`, which is called from the scheduler (26). This function receives two arguments:

*   The address of the current process's stack pointer, where the pointer itself will be stored after saving the current process's context (24).
*   The stack pointer of the next process (23).

When implementing the low-level context switch function, attention must be paid to the calling conventions and parameter passing for the specific platform and compiler.

#### Scheduler with Software Interrupt

In this variant, the scheduler differs significantly from the one described above. The main difference is that the actual context switch does not occur via a direct call to the context switcher function but by activating a special interrupt, within which the context switch takes place. This method harbors several nuances and requires special measures to prevent disruption of the system's integrity.

The primary difficulty in implementing this control transfer method is that the scheduler code itself and the software interrupt handler code are not strictly continuous or "atomic"; an interrupt can occur between them, which could also initiate rescheduling. This would cause a kind of "overlap" of the current rescheduling results and disrupt the integrity of the control transfer process. To avoid this conflict, the "rescheduling-control transfer" process is split into two "atomic" operations that can be safely separated from each other.

The first operation is, as before, calculating the priority of the highest-priority ready process — calling the `update_sched_prio()` function (01) — and checking the need for rescheduling (32). If such a need exists, the priority value of the next process is recorded in the `SchedProcPriority` variable (07), and the context switch software interrupt is activated (34). The program then enters a loop waiting for the context switch (35).

A rather subtle point is hidden here. One might ask, why not simply implement the enabled-interrupts zone as a pair of dummy instructions (to give the processor hardware time to actually trigger the interrupt)? Such an implementation contains a subtle error, which consists of the following.

If, at the moment when context switching is enabled (which in this OS version is implemented by globally enabling interrupts (37)), one or several other interrupts besides the software interrupt were pending, and the priority of some of them is higher than that of the software interrupt, then control will naturally be transferred to the handler of that higher-priority interrupt. Upon its completion, a return will be made to the interrupted program. Now, in the main program (i.e., inside the scheduler function), the processor may execute one or several instructions[^8] before the next interrupt can be activated.

[^8]: This is a common property of many processors — after returning from an interrupt, a jump to the handler of the next interrupt is not possible immediately in the same machine cycle, but only after one or more cycles.

In this case, the program might reach the code that disables context switching, which would lead to interrupts being globally disabled, and the software interrupt, where the context switch is performed, would not be executed. This means the execution flow would remain in the current process, while it should have been transferred to the system (and other processes) until an event that the current process is waiting for occurs. This is nothing other than a breach of system integrity and can lead to a wide variety of difficult-to-predict negative consequences.

Obviously, such a situation must not arise. Therefore, instead of a few dummy instructions in the enabled-interrupts zone, a loop waiting for the context switch is used. That is, no matter how many interrupts are queued, until an actual context switch occurs, the program's execution flow will not proceed beyond this loop.

For the described mechanism to work, a criterion is needed to determine that rescheduling has indeed occurred. Such a criterion can be the equality of the kernel variables `CurProcPriority` and `SchedProcPriority`. These variables become equal to each other (i.e., the value of the current priority becomes equal to the scheduled value) only after the context switch has been performed.

As can be seen, there are no updates here to variables containing stack pointers or the current priority value. All these actions are performed later during the actual context switch by calling a special kernel function `os_context_switch_hook()`.

One might wonder: why such complexity? To answer this question, consider a scenario: suppose, in the case of context switching via a software interrupt, the scheduler implementation remained the same as in the case of a direct call to the context switcher. But the call:

```cpp
os_context_switcher(Curr_SP_addr, Next_SP);
```

is replaced by[^9]:

```cpp
raise_context_switch();
<wait_for_context_switch_done>;
```

[^9]: `<wait_for_context_switch_done>` implies all the code that ensures context switching from starting of interrupts enabling.

Now, consider a situation where, at the moment interrupts are enabled, one or several other interrupts are pending, and at least one of them has a higher priority than the software interrupt for context switching. Furthermore, in the handler of this higher-priority pending interrupt, a service function (an inter-process communication facility) is called. What would happen in this case?

In this case, the scheduler would be called again, and another process rescheduling would occur. However, since the previous rescheduling was not completed—i.e., the processes did not actually switch, and contexts were not physically saved and restored—the new rescheduling would simply overwrite the variables containing the pointers to the current and next processes.

Moreover, when determining the need for rescheduling, the value of `CurProcPriority` would be used, which is actually incorrect because this value represents the priority of the next process scheduled during the previous scheduler call. In short, a "overlap" of scheduling operations would occur, disrupting the integrity of the system's operation.

Therefore, it is crucial that the actual update of the `CurProcPriority` value and the switching of process contexts are "atomic"—indivisible and not interrupted by any other code related to process scheduling. In the variant with direct context switcher call, this rule is inherently satisfied—the entire scheduler's work happens within a critical section, and the context switcher is called directly from there.

In the software interrupt variant, scheduling and context switching can be "separated" in time. Therefore, the actual context switch and the change of the current priority occur directly during the execution of the software interrupt handler[^10]. Immediately after saving the context of the current process, the `os_context_switch_hook()` function is called (where the `CurProcPriority` value is directly updated). Also, the stack pointer of the current process is passed to `os_context_switch_hook()`, where it is saved in the current process object, and the stack pointer of the next process, required for restoring its context and subsequently transferring control to it, is retrieved and returned from the function.

To avoid degrading performance in interrupt handlers, there is a special lightweight, inline version of the scheduler used by some member functions of service objects optimized for use in ISRs. See "Listing 4. Scheduler Variant Optimized for Use in ISR" for the code of this scheduler version.

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
Listing 4. Scheduler Variant Optimized for Use in ISR
///

When selecting the context switch interrupt handler, preference should be given to the one with the lowest priority (in the case of a priority interrupt controller). This helps avoid unnecessary rescheduling and context switches when several interrupts occur in succession.

### Pros and Cons of Control Transfer Methods

Both methods have their advantages and disadvantages. The advantages of one control transfer method are the disadvantages of the other, and vice versa.

#### Direct Control Transfer

The main advantage of direct control transfer is that implementing this variant does not require a dedicated software interrupt in the target MCU—not all MCUs have such hardware capability. A second minor advantage is slightly higher performance compared to the software interrupt variant, as the latter incurs additional overhead for activating the context switch interrupt handler, the loop waiting for context switch, and the call to `os_context_switch_hook()`.

The direct control transfer variant has a serious drawback—when the scheduler is called from an interrupt handler, the compiler is forced to save the "local context" (the processor's scratch registers) due to the call to a non-inline context switch function. This overhead can be quite significant compared to the rest of the ISR code. The negative aspect here is that saving these registers may be completely unnecessary—since the function[^11] that necessitates their saving does not use these registers. Therefore, if there are no more calls to non-inline functions, the code for saving and restoring this register group becomes redundant.

[^11]:
```cpp
os_context_switcher(stack_item_t **Curr_SP, stack_item_t *Next_SP)
```

#### Software Interrupt-Based Control Transfer

This variant is free from the drawback described above. Because the ISR itself executes in the usual manner and no rescheduling is performed from within it, the saving of the "local context" is also not performed. This significantly reduces overhead and improves system performance. To avoid spoiling this advantage by calling a non-inline member function of an inter-process communication service object, it is recommended to use special, lightweight, inline versions of such functions—see the [Inter-Process Communication Facilities](ipcs.md) section for more details.

The main disadvantage of control transfer using a software interrupt is that not all hardware platforms support a software interrupt. In such cases, one of the unused hardware interrupts can be used as this software interrupt. Unfortunately, this introduces a lack of universality—it is not known in advance whether a particular hardware interrupt will be required in a given project. Therefore, if the processor does not specifically provide a suitable interrupt, the choice of the context switch interrupt is delegated (from the port level) to the project level, and the user must write the corresponding code themselves[^12].

[^12]: The **scmRTOS** distribution is offered with several working usage examples where all this code for organizing and configuring the software interrupt is present. Therefore, the user can simply modify this code for their project's needs or use it as-is if it suits them.

When using control transfer via a software interrupt, the expression "The kernel takes control away from the processes" fully reflects the situation.

#### Conclusions

Considering the above analysis of the advantages and disadvantages of both control transfer methods, the general recommendation is as follows: if the target platform provides a suitable interrupt for implementing context switching, it makes sense to use this variant, especially if the size of the "local context" is substantial.

Using direct control transfer is justified when using a software interrupt is practically impossible—for example, when the target platform does not support such an interrupt, or using a hardware interrupt as a software one is not feasible for various reasons. It may also be chosen if performance characteristics with this transfer method are better due to lower overhead for organizing context switches, and the saving/restoring of the "local context" does not incur significant overhead because of its small size[^13].

[^13]: For example, on **MSP430**/IAR, the "local context" consists of only 4 registers.

### Support for Inter-Process Communication

Support for inter-process communication (IPC) boils down to providing a set of functions for monitoring process states and granting IPC facilities access to the rescheduling mechanisms of the OS. For more details, see the [Inter-Process Communication Facilities](ipcs.md) section.

### Interrupts

#### Usage Peculiarities with an RTOS and Implementation

An occurred interrupt can be the source of an event that needs to be handled by a specific process. Therefore, to minimize (and make deterministic) the response time to an event, process rescheduling and control transfer to the highest-priority ready process are used when necessary.

The code of any interrupt handler that uses IPC services must call the `isr_enter()` function upon entry, which increments the `ISR_NestCount` variable, and call the `isr_exit()` function upon exit. The latter decrements `ISR_NestCount` and uses its value to determine the interrupt nesting level (in case of nested interrupts). When `ISR_NestCount` becomes 0, it indicates a return from the interrupt handler to the main program. At this point, `isr_exit()` performs process rescheduling (if needed) by calling the interrupt-level scheduler.

To simplify usage and ensure portability, the code executed upon entry and exit of interrupt handlers is placed in the constructor and destructor, respectively, of a special wrapper class—`TISRW`. An object of this type must be used within the interrupt handler[^14]. It is sufficient to create an object of this type in the interrupt handler code; the compiler will handle the rest. It is important that the declaration of this object comes before the first use of any service functions.

[^14]: The aforementioned functions `isr_enter()` and `isr_exit()` are member functions of this wrapper class.

It should be noted that if a non-inline function is called from within an interrupt handler, the compiler will save the "local context"—the scratch[^15] registers[^16]. Therefore, it is desirable to avoid calls to non-inline functions from interrupt handlers, as even partial context saving degrades both speed and code size[^17]. In light of this, the current version of **scmRTOS** includes special additional lightweight functions for some IPC objects intended for use within interrupt handlers. These functions are inline and use a lightweight version of the scheduler, which is also inline. For more details, see the [Inter-Process Communication Facilities](ipcs.md) section.

[^15]: Typically, the compiler divides processor registers into two groups: scratch and preserved. Scratch registers are those that any function can use without prior saving. Preserved registers are those whose values must be saved if needed. In some contexts, preserved registers are called local; in the context discussed here, these terms are synonymous.
[^16]: On different platforms, the proportion (of the total number) of these registers varies. For example, when using EWAVR, they occupy about half of the total, while with EW430, it's less than half. In the case of VisualDSP++/**Blackfin**, the proportion of these registers is large, but on this platform, stack sizes are typically sufficiently large not to worry about it.
[^17]: Unfortunately, when using the direct control transfer scheme, a call to the non-inline context switch function occurs, making it impossible to avoid the overhead of saving scratch registers in this case.

#### Separate Interrupt Stack and Nested Interrupts

Another aspect of using a preemptive RTOS is related to interrupts. As known, when an interrupt occurs and control is transferred to its handler, the program uses the stack of the interrupted process for its execution. This stack must have a size sufficient to meet the needs of both the process itself and any interrupt handler. Moreover, it must account for the combined worst-case scenario—for instance, when process execution occupies the peak stack space, and an interrupt occurs whose handler also consumes part of the stack. The stack size must be such that overflow does not occur even in this case.

Obviously, the above circumstances concern all processes in the system. If interrupt handlers consume a significant amount of stack space, the stack sizes of all processes must be increased by a certain amount. This leads to increased memory overhead. In the case of nested interrupts, the situation worsens dramatically.

To counter this effect, switching the processor's stack pointer to a dedicated interrupt stack upon the occurrence of an interrupt is employed. This way, process stacks and the interrupt stack become "decoupled" from each other, eliminating the need to reserve additional memory in each process's stack for interrupt handler execution.

The implementation of a separate interrupt stack is done at the port level. Some processors have hardware support for switching the stack pointer to an interrupt stack, making the use of this capability efficient and safe[^18].

[^18]: In this case, this mechanism is the only one implemented in the port, and there is no need for a separate implementation of the `TISRW_SS` wrapper class.

Nested interrupts—i.e., those whose handlers can interrupt not only the main program but also other interrupt handlers—also have specific usage characteristics. Understanding these is important for the effective and safe use of this mechanism. In the case of a processor with an interrupt controller supporting multi-level prioritized interrupts, the situation with nested interrupts is relatively straightforward—potential hazardous situations when enabling nested interrupts are generally accounted for by the processor designers, and the interrupt controller prevents mishaps, such as those described below.

In the case of a processor with a single-level interrupt system, its implementation typically involves automatically disabling interrupts globally upon the occurrence of any interrupt. This is done for simplicity and safety. That is, nested interrupts are not supported in such a system. To enable nested interrupts, it is sufficient to perform a global enable interrupts operation, which on processors with a single-level interrupt system is typically disabled by hardware when control is transferred to an interrupt handler. In this scenario, a situation is possible where an already executing interrupt handler is called again—if a request for handling the same interrupt is still pending[^19].

[^19]: This could be related, for example, to events triggering the interrupt too frequently or an uncleared interrupt flag that initiates the interrupt request.

Typically, this is an erroneous situation that must be avoided. To prevent finding oneself in such a position, one must clearly understand both the processor's operational specifics and its "context"[^20], and write code very carefully: before globally enabling interrupts, disable the activation of the interrupt whose handler is already executing to avoid re-entry into the same handler. Upon completion of its work, do not forget to restore the processor's control resources to their original state, as it was before manipulating nested interrupt enabling.

Based on the above, the following recommendation can be given.

[^20]: Here, "context" implies the logical and semantic environment in which this part of the program is executing.

!!! error "**WARNING**"

    Despite the apparent advantage of the separate interrupt stack scheme, using this variant on processors that lack hardware support for switching the stack pointer to an interrupt stack is **not recommended**.

    This is due to the additional overhead of stack switching, poor portability—any non-standard extensions are a source of problems—and the fact that directly interfering with stack pointer management can, in one way or another, cause conflicts with local object addressing. For example, the compiler, seeing the body of an interrupt handler, allocates[^21] memory for local objects on the stack. Moreover, it does this *before* the call[^22] to the wrapper's constructor. Thus, after switching the stack pointer to the interrupt stack, the memory allocated earlier will physically be in a different location. The program will then operate incorrectly, and the compiler will be unable to detect this situation.

    Similarly, using nested interrupts on processors that do not support this capability in hardware is **not recommended**. Such interrupts require careful handling and typically need additional management—for instance, locking the interrupt source to prevent another call to the same handler when interrupts are enabled.

[^21]: More precisely — reserves it. This is usually done by modifying the stack pointer.
[^22]: It has every right to do so.

**Brief conclusion.** The motivation for using stack pointer switching to an interrupt stack correlates with the use of nested interrupts—after all, in the case of nested interrupts, stack consumption (within interrupts) increases significantly. This imposes—in the absence of switching to a separate interrupt stack—additional requirements on the stack sizes of all processes[^23].

!!! tip "**TIP**"

    When using a preemptive RTOS, it is possible to structure the program so that interrupt handlers serve only as **event sources**, and all event processing is moved to the process level. This allows interrupt handlers to be small and fast, which, in turn, eliminates the need for both switching to an interrupt stack and enabling nested interrupts. In this case, the body of the interrupt handler can be comparable in size to the overhead of switching the stack pointer to an interrupt stack and enabling nested interrupts.

[^23]: Furthermore, **each** process must have a stack size large enough to cover both the process's own needs and the stack consumption of interrupt handlers, including the entire nesting hierarchy.

This is precisely the recommended approach when the processor lacks hardware support for switching the stack pointer to an interrupt stack and does not have an interrupt controller with hardware support for nested interrupts.

It should be noted that a priority-based preemptive RTOS is, in a way, an analogue of a multi-level priority interrupt controller. That is, it provides the ability to distribute code execution according to importance/urgency. In this regard, in most cases, there is no need to place event handling code at the interrupt level, even if such a hardware controller exists. Instead, interrupts should be used only as **event sources**[^24], with their handling placed at the process level. This is the recommended program design style.

[^24]: By making interrupt handlers as simple, short, and fast as possible.

### System Timer

The system timer is used to generate specific time intervals required for process operation. This includes timeout support.

Typically, one of the processor's hardware timers is used as the system timer[^25].
The functionality of the system timer is implemented in the kernel function `system_timer()`. See "Listing 5. System Timer" for the code of this function.

[^25]: The simplest timer (without "bells and whistles") is suitable for this. The only fundamental requirement is that it must be capable of generating periodic interrupts at equal intervals—for example, an overflow interrupt. It is also desirable to have the ability to control the overflow period to select a suitable system tick frequency.

```cpp
01    void OS::TKernel::system_timer()
02    {
03        SYS_TIMER_CRIT_SECT();
04    #if scmRTOS_SYSTEM_TICKS_ENABLE == 1
05        SysTickCount++;
06    #endif
07    
08    #if scmRTOS_PRIORITY_ORDER == 0
09        const uint_fast8_t BaseIndex = 0;
10    #else
11        const uint_fast8_t BaseIndex = 1;
12    #endif
13    
14        for(uint_fast8_t i = BaseIndex; i < (PROCESS_COUNT-1 + BaseIndex); i++)
15        {
16            TBaseProcess *p = ProcessTable[i];
17    
18            if(p->Timeout > 0)
19            {
20                if(--p->Timeout == 0)
21                {
22                    set_process_ready(p->Priority);
23                }
24            }
25        }
26    }
```
/// Caption
Listing 5. System Timer
///

As can be seen from the source code, the actions are very simple:

  1. If the tick counter is enabled, the counter variable is incremented (5).
  2. Then, in a loop, the timeout values of all registered processes are checked. If the value of the variable being checked is not equal to 0[^26], it is decremented and checked for 0. If it equals 0 (after decrementing)—i.e., the timeout for this process has expired—the process is moved to the ready state.

[^26]: This means the process is waiting with a timeout.

Since this function is called inside the timer interrupt handler, upon returning to the main program (as described above), control will be transferred to the highest-priority process ready for execution. That is, if the timeout of some process (with higher priority than the interrupted one) expires, it will gain control upon exiting the interrupt. This is implemented using the scheduler (see above)[^27].

[^27]: Given the ability to set the priority of the system timer interrupt.

!!! info "**NOTE**"

    Some operating systems provide recommendations for setting the system tick duration. Most often, a range of 10[^28] to 100 ms is mentioned. Perhaps for those operating systems, this is correct. The balance here is determined by the desire to achieve the lowest overhead from system timer interrupts and the desire to obtain finer time resolution.

    Given **scmRTOS**'s orientation towards small MCUs operating in real-time, and taking into account the fact that the (execution time) overhead[^29] is small, the recommended system tick value is **1 to 10 ms**.

    An analogy can be drawn with other domains where smaller objects are usually higher-frequency: for example, a mouse's heartbeat is much faster than a human's, and a human's is faster than an elephant's. At the same time, "agility" is inversely related. A similar trend exists in technology; therefore, it is reasonable to expect that for small processors, the system tick period is smaller than for large ones—in larger systems, overhead is generally greater due to the typically higher load on more powerful processors and, consequently, their lower "agility".

[^28]: How, for example, can dynamic indication be organized with such a digit switching period when it is known that for comfortable operation, the switching period (for four digits) should be no more than 5 ms?

[^29]: Due to the small number of processes and the simple, fast scheduler.

----

<a name="kernel-agent"></a>
## TKernelAgent and Extensions

### Kernel Agent

The `TKernelAgent` class is a special facility designed to provide access to kernel resources when building extensions to enhance the operating system's functionality.

The general idea is as follows. Creating various functional extensions for the OS requires access to certain kernel resources—in particular, to the kernel variable containing the priority of the active process or to the system's process map. Providing direct access to this part of the internal representation would not be very wise—it would violate the security model of the object-oriented approach[^30]. This leads to negative consequences, such as program inoperability in the absence of proper coding discipline and/or loss of compatibility in case of changes to the kernel's internal representation.

Therefore, to solve the problem of accessing kernel resources, an approach based on a specially created class—the kernel agent—has been proposed. This class restricts access through its documented interface. All this allows for the creation of extensions in a formalized way, making the process simpler and safer.

[^30]: The principles of encapsulation and abstraction.

The code for the kernel agent class can be found in "Listing 6. TKernelAgent".

```cpp
01    class TKernelAgent
02    {
03        INLINE static TBaseProcess * cur_proc() { return Kernel.ProcessTable[cur_proc_priority()]; }
04
05    protected:
06        TKernelAgent() { }
07        INLINE static uint_fast8_t const   & cur_proc_priority()       { return Kernel.CurProcPriority;  }
08        INLINE static volatile TProcessMap & ready_process_map()       { return Kernel.ReadyProcessMap;  }
09        INLINE static volatile timeout_t   & cur_proc_timeout()        { return cur_proc()->Timeout;     }
10        INLINE static void reschedule()                                { Kernel.scheduler();             }
11
12        INLINE static void set_process_ready   (const uint_fast8_t pr) { Kernel.set_process_ready(pr);   }
13         INLINE static void set_process_unready (const uint_fast8_t pr) { Kernel.set_process_unready(pr); }
14 
15     #if scmRTOS_DEBUG_ENABLE == 1
16         INLINE static TService * volatile & cur_proc_waiting_for()     { return cur_proc()->WaitingFor;  }
17     #endif
18 
19     #if scmRTOS_PROCESS_RESTART_ENABLE == 1
20         INLINE static volatile 
21         TProcessMap * & cur_proc_waiting_map()  { return cur_proc()->WaitingProcessMap; }
22     #endif
23     };                                                                                         
```                                
/// Caption
Listing 6. TKernelAgent
///

As can be seen from the code, the class is defined in such a way that it is impossible to create objects of this class. This is done deliberately because, by design, `TKernelAgent` serves as a base for creating extensions: its main function is to provide a documented interface to kernel resources. Therefore, all use of this code becomes possible only in descendants of this class, which are the actual extensions.

An example of using `TKernelAgent` will be examined in more detail later when describing the base class for creating interprocess communication services, `TService`.

The entire class interface consists of inline functions, which in most cases allows implementing the necessary extensions without loss of efficiency compared to the scenario where access to kernel resources is performed directly.

### Extensions

The aforementioned kernel agent class enables the creation of additional facilities that extend the functional capabilities of the OS. The methodology for creating such facilities is simple—just declare a class that inherits from `TKernelAgent` and define its contents. Such classes are called operating system extensions.

The placement of the OS kernel code is such that the definitions of classes and a number of member function definitions are separated in the header file `os_kernel.h`. This makes it possible to write a user class that has access to all definitions of OS kernel types, while at the same time, the definitions of this user class become available in the member functions of the kernel classes—for example, in the scheduler and in the system timer function[^31].

[^31]: In user hooks.

Extensions are connected using the configuration file `scmRTOS_extensions.h`, which is included in `os_kernel.h` between the definitions of kernel types and their member functions. This allows the definition of an extension class to be physically placed in a separate user header file and included in the project by including this file in `scmRTOS_extensions.h`. After this, the extension is ready for use according to its intended purpose.

