# Interprocess Communication Services

----

## Introduction

Starting with version 4, **scmRTOS** employs a fundamentally different mechanism for implementing interprocess communication services compared to previous versions. Previously, each service class was developed individually and had no connection to the others, and to access kernel resources, service classes were declared as "friends" of the kernel. This approach did not allow for code reuse[^1] and offered no possibility to extend the set of services, which led to the decision to abandon it and design a variant free from both shortcomings.

[^1]: Since interprocess communication services perform similar actions when interacting with kernel resources, they contain, in places, nearly identical code.

The implementation is based on the concept of extending OS functionality by defining extension classes through inheritance from `TKernelAgent` (see ["TKernelAgent and Extensions"](kernel.md#kernel-agent)).

The key class for building interprocess communication services is the `TService` class, which implements the common functionality for all service classes. All of them are descendants of `TService`. This applies both to the standard set of services included in the OS distribution and to those developed[^2] as extensions to the standard range of services.

[^2]: Or can be developed by the user for their project's needs.

The interprocess communication services included in scmRTOS are:

*   `OS::TEventFlag`;
*   `OS::TMutex`;
*   `OS::message`;
*   `OS::channel`;

----

## TService

### Class Definition

The code for the base class used to build service types:

```cpp
01    class TService : protected TKernelAgent
02    {
03    protected:
04        TService() : TKernelAgent() { }
05
06        INLINE static TProcessMap  cur_proc_prio_tag()  { return get_prio_tag(cur_proc_priority()); }
07        INLINE static TProcessMap  highest_prio_tag(TProcessMap map)
08        {
09        #if scmRTOS_PRIORITY_ORDER == 0
10            return map & (~static_cast<unsigned>(map) + 1);   // Isolate rightmost 1-bit.
11        #else   // scmRTOS_PRIORITY_ORDER == 1
12            return get_prio_tag(highest_priority(map));
13        #endif
14        }
15
16        //----------------------------------------------------------------------
17        //
18        //   Base API
19        //
20
21        // add prio_tag proc to waiters map, reschedule
22        INLINE void suspend(TProcessMap volatile & waiters_map);
23
24        // returns false if waked-up by timeout or TBaseProcess::wake_up() | force_wake_up()
25        INLINE static bool is_timeouted(TProcessMap volatile & waiters_map);
26
27        // wake-up all processes from waiters map
28        // return false if no one process was waked-up
29               static bool resume_all     (TProcessMap volatile & waiters_map);
30        INLINE static bool resume_all_isr (TProcessMap volatile & waiters_map);
31
32        // wake-up next ready (most priority) process from waiters map if any
33        // return false if no one process was waked-up
34               static bool resume_next_ready     (TProcessMap volatile & waiters_map);
35        INLINE static bool resume_next_ready_isr (TProcessMap volatile & waiters_map);
36    };
```        
/// Caption
Listing 1. TService
///

Similar to its ancestor class `TKernelAgent`, the `TService` class does not allow objects of its own type to be created: its purpose is to provide a base for building specific types—interprocess communication services. The interface of this class consists of a set of functions that express the common actions of any service class within the context of control transfer between processes. Logically, these functions can be divided into two parts: core functions and utility functions.

The utility functions are:

1.  `TService::cur_proc_prio_tag()`. Returns the tag[^3] corresponding to the currently active process. This tag is actively used by the core service functions to record the identifiers[^4] of processes when placing the current process into a waiting state.
2.  `TService::highest_prio_tag()`. Returns the tag of the highest-priority process from the process map passed as an argument. It is used primarily to obtain the identifier (of the process) from the process map of a service object, corresponding to the process that should be transitioned to the ready state.

[^3]: A process tag is technically a mask of type `TProcessMap` with only one non-zero bit. The position of this bit in the mask corresponds to the process priority. Process tags are used for manipulating `TProcessMap` objects, which define the ready/not-ready state of processes, and also serve to record process tags.
[^4]: Along with the process priority number, the tag can also serve as a process identifier—there is a one-to-one correspondence between the priority number and the process tag. Each type of identifier has advantages in terms of efficiency in specific situations, so both types are used extensively in the OS code.

The core functions are:

1.  `TService::suspend()`. Transfers the process to a not-ready state, records the process identifier in the service's process map, and calls the OS scheduler. This function forms the basis of the service member functions used for waiting for an event (`wait()`, `pop()`, `read()`) or actions that might cause waiting for a resource to be released (`lock()`, `push()`, `write()`).
2.  `TService::is_timeouted()`. This function returns `false` if the process was transitioned to the ready state by calling a service member function; however, if the process was transitioned to the ready state due to a timeout[^5] or forcibly using the `TBaseProcess` member functions `wake_up()` and `force_wake_up()`, then the function returns `true`. The result of this function is used to determine whether the process received the event (resource release) it was waiting for or not.
3.  `TService::resume_all()`. This function checks for the presence of processes "recorded" in the service's process map but currently in a not-ready state[^6]; if such processes exist, all of them are transitioned to the ready state and the scheduler is called. In this case, the function returns `true`; otherwise, it returns `false`.
4.  `TService::resume_next_ready()`. This function performs actions similar to the `resume_all()` function described above, with the difference that if there are waiting processes, not all of them are transitioned to the ready state, but only one—the highest-priority one.

[^5]: In other words, "awakened" in the system timer interrupt handler.
[^6]: That is, processes whose waiting state has not been interrupted by a timeout and/or forcibly using `TBaseProcess::wake_up()` and `TBaseProcess::force_wake_up()`.

For the `resume_all()` and `resume_next_ready()` functions, there are versions optimized for use inside interrupt handlers—these are `resume_all_isr()` and `resume_next_ready_isr()`. In purpose and meaning, they are similar to the main variants[^7]; the key difference is that they do not call the scheduler.

[^7]: Therefore, a full description of them is not provided.

### Usage

#### Preliminary Remarks

Any service class is created from `TService` through inheritance. To illustrate the usage of `TService` and the creation of a service class based on it, we will examine one of the standard interprocess communication services—`TEventFlag`:

```cpp
class TEventFlag : public TService { ... }
```

The service class `TEventFlag` itself will be described in detail later. For the continuity of the narrative at this point, it should be noted that this interprocess communication service is used to synchronize the work of processes in accordance with occurring events. The main idea of its usage is: one process waits for an event using the class member function `TEventFlag::wait()`, while another process[^8], upon the occurrence of an event that should be handled by the waiting process, signals the flag using the member function `TEventFlag::signal()`.

[^8]: Or an interrupt handler—depending on the source of the events. For interrupt handlers, there is a special version of the function that signals the flag, but in the context of the current description, this nuance is omitted as non-essential.

Given the above, the main focus when examining the usage example will be on these two functions, as they carry the primary semantic load of the service class[^9], and its development essentially boils down to the development of such functions.

[^9]: The rest of its representation is auxiliary in nature and serves to give completeness to the class and improve its user characteristics.

#### Requirements for the Functions of the Class Being Developed

Requirements for the event flag waiting function. The function must check whether the event has occurred at the moment of the function call. If the event has not occurred, it must be able to wait[^10] for the event either unconditionally or with a timeout condition. If the function returns due to the event, the return value should be `true`; if it returns due to a timeout, the return value should be `false`.

[^10]: I.e., yield control to the system kernel and remain in passive waiting.

Requirements for the event flag signaling function. The function must transition all processes waiting for the event flag to the ready state and transfer control to the scheduler.

#### Implementation

Inside the member function `wait()` (see "Listing 2. Function TEventFlag::wait()"), the first step is to check if the event has already been signaled. If it has, the function returns `true`. If the event has not been signaled (i.e., it needs to be waited for), preparatory actions are performed—in particular, the waiting timeout value is written to the `Timeout` variable of the current process, and the `suspend()` function, defined in the base class `TService`, is called. This function writes the tag of the current process into the process map of the event flag object (passed to `suspend()` as an argument), transitions this process to a not-ready state, and yields control to other processes by calling the scheduler.

Upon returning from `suspend()`, which means this process has been transitioned back to the ready state, a check is performed to determine what caused the "wake-up" of this process. This is done by calling the `is_timeouted()` function. It returns `false` if the process was "awakened" via a call to `TEventFlag::signal()`—i.e., the awaited event occurred (and no timeout happened). It returns `true` if the process "wake-up" occurred due to the expiration of the timeout specified in the `TEventFlag::wait()` argument, or if it was forced.

This logic of the `TEventFlag::wait()` member function allows it to be effectively used in user code when organizing process work synchronized with the occurrence of required events[^11]. Moreover, the implementation code of this function is simple and transparent.

[^11]: Including when such events do not occur within the specified time interval.

```cpp
01    bool OS::TEventFlag::wait(timeout_t timeout)
02    {
03        TCritSect cs;
04   
05        if(Value)                         // if flag already signaled
06        {
07            Value = efOff;                // clear flag
08            return true;
09        }
10        else
11        {
12            cur_proc_timeout() = timeout;
13   
14            suspend(ProcessMap);
15   
16            if(is_timeouted(ProcessMap))
17                return false;            // waked up by timeout or by externals
18   
19            cur_proc_timeout() = 0;
20            return true;                 // otherwise waked up by signal() or signal_isr()
21        }
22    }
```
/// Caption
Listing 2. Function TEventFlag::wait()
///

```cpp
1    void OS::TEventFlag::signal()
2    {
3        TCritSect cs;
4        if(!resume_all(ProcessMap))   // if no one process was waiting for flag
5            Value = efOn;
6    }
```
/// Caption
Listing 3. Function TEventFlag::signal()
///

The code for the `TEventFlag::signal()` function (see "Listing 3. Function TEventFlag::signal()") is extremely simple: inside it, all processes waiting for this event flag are transitioned to the ready state, and rescheduling is performed. If no such processes were found, the internal flag variable `Value` is set to `efOn` (true), which means the event occurred but hasn't been handled by anyone yet.

Any interprocess communication service can be designed and defined in a similar manner. During its development, it is only necessary to have a clear understanding of what the member functions of the `TService` class do and to use them appropriately.

----

## OS::TEventFlag

During program execution, there is often a need for synchronization between processes. For example, one of the processes may need to wait for an event to perform its work. It can handle this in different ways: it can simply poll a global flag in a loop or do the same with some periodicity—i.e., poll, "go to sleep" with a timeout, "wake up," poll again, etc. The first method is bad because all processes with lower priority will not gain control, as due to their lower priorities, they cannot preempt the process polling the global flag in a loop.

The second method is also bad—the polling period becomes quite large (i.e., time resolution is low), and during polling, the process will occupy the CPU with context switches, even though it is unknown whether the event has occurred.

A proper solution in this situation is to transition the process to a state of waiting for the event and transfer control to the process only when the event occurs.

This functionality in scmRTOS is implemented using `OS::TEventFlag` objects (event flag). The class definition is shown in "Listing 4. OS::TEventFlag".

```cpp
01    class TEventFlag : public TService                                            
02    {                                                                             
03    public:                                                                       
04        enum TValue { efOn = 1, efOff= 0 }; // prefix 'ef' means: "Event Flag"
05                                                                                  
06    public:                                                                       
07        INLINE TEventFlag(TValue init_val = efOff);
08                                                                                  
09               bool wait(timeout_t timeout = 0);                                  
10        INLINE void signal();                                                     
11        INLINE void clear()       { TCritSect cs; Value = efOff; }                
12        INLINE void signal_isr();                                                 
13        INLINE bool is_signaled() { TCritSect cs; return Value == efOn; }         
14                                                                                  
15    private:                                                                      
16        volatile TProcessMap ProcessMap;                                          
17        volatile TValue      Value;                                               
18    };                                                                            
```
/// Caption
Listing 4. OS::TEventFlag
///

### Interface

---- 

#### <u>wait</u>

###### Prototype

```cpp
bool OS::TEventFlag::wait(timeout_t timeout);
```

###### Description

Waits for an event. When the `wait()` function is called, the following occurs: it checks if the flag is set. If it is set, the flag is cleared and the function returns `true`, meaning the event had already occurred at the time of the check. If the flag is not set (i.e., the event has not yet occurred), the process is transitioned to a state of waiting for this flag (event) and control is yielded to the kernel, which, after rescheduling processes, will run the next one.

If the function is called without arguments (or with an argument equal to 0), the process will remain in the waiting state until the event flag is "signaled" by another process or an interrupt handler (using the `signal()` function) or is brought out of the inactive state using the `TBaseProcess::force_wake_up()` function (in the latter case, extreme caution is required).

If `wait()` is called without an argument, it always returns `true`. If the function is called with an argument (an integer greater than 0) specifying the waiting timeout in system timer ticks, the process will wait for the event as in the case of a call without an argument. However, if the event flag is not "signaled" within the specified period, the process will be "awakened" by the timer and the function will return `false`. This implements both unconditional waiting and waiting with a timeout.

---- 

#### <u>signal</u>

###### Prototype

```cpp
INLINE void OS::TEventFlag::signal();
```

###### Description

A process that wishes to notify other processes via a `TEventFlag` object that a particular event has occurred must call the `signal()` function. As a result, all processes waiting for the specified event will be transitioned to the ready state, and the highest-priority among them will gain control (the others will follow in order of their priorities).

---- 

#### <u>signal from ISR</u>

###### Prototype

```cpp
INLINE void OS::TEventFlag::signal_isr();
```

###### Description

A variant of the above function optimized for use in interrupts. The function is inline and uses a special lightweight inline version of the scheduler. This variant must not be used outside of interrupt handler code.

---- 

#### <u>clear</u>

###### Prototype

```cpp
INLINE void OS::TEventFlag::clear();
```

###### Description

Clears the flag. Sometimes, for synchronization, it is necessary to wait for the *next* event, not to handle an already occurred one. In this case, the event flag must be cleared before proceeding to wait. The `clear()` function is used for this purpose.

---- 

#### <u>check if signaled</u>

###### Prototype

```cpp
INLINE bool OS::TEventFlag::is_signaled();
```

###### Description

Checks the state of the flag. It is not always necessary to wait for an event by yielding control. Sometimes, based on the program's logic, it is only necessary to check whether the event has occurred.

---- 

### Usage Example

An example of using an event flag is shown in "Listing 5. Using TEventFlag".

In this example, process `Proc1` waits for an event with a timeout of 10 system timer ticks (9). The second process, `Proc2`, "signals" the flag when a condition is met (27). If the first process has higher priority, it will gain control immediately.

```cpp
01    OS::TEventFlag eflag;
02    ...
03    //----------------------------------------------------------------
04    template<> void Proc1::exec()
05    {
06        for(;;)
07        {
08            ...
09            if( eflag.wait(10) ) // wait event for 10 ticks
10            {
11                ...   // do something
12            }
13            else
14            {
15                ...   // do something else
16            }
17            ...
18        }
19    }
20    ...
21    //----------------------------------------------------------------
22    template<> void Proc2::exec()
23    {
24        for(;;)
25        {
26            ...
27            if( ... ) eflag.signal(); 
28            ...
29        }
30    }
31    //----------------------------------------------------------------
```
/// Caption
Listing 5. Using TEventFlag
///

!!! info "**NOTE**"

    When an event occurs and a process "signals" the flag, **all** processes that were waiting for this flag will be transitioned to the ready state. In other words, everyone who was waiting has now received the event. They will, of course, gain control in the order of their priorities, but the event will not be missed by any process that managed to start waiting, regardless of its priority.

    Thus, an event flag *exhibits broadcast behavior*, which is very useful for organizing notifications and synchronizing multiple processes based on a single event. Of course, nothing prevents using an event flag in a "point-to-point" scheme where there is only one process waiting for the event.

----

## OS::TMutex

The Mutex semaphore (from Mutual Exclusion), as the name suggests, is designed to organize mutual exclusion of access. That is, at any given moment, no more than one process can hold this semaphore. If any process attempts to lock a Mutex that is already held by another process, the attempting process will wait until the semaphore is released.

The primary use of Mutex semaphores is to organize mutual exclusion when accessing a particular resource. For example, a static array with global visibility[^12] is used by two processes to exchange data. To avoid errors during the exchange, it is necessary to prevent one process from having access to the array during the period when another process is working with it.

Using a critical section for this is not the best approach, as interrupts would be disabled for the entire time the process accesses the array. This time might be significant, and during it, the system would be unable to respond to events. In this situation, a mutual exclusion semaphore is perfectly suitable: a process planning to work with a shared resource must first lock the Mutex semaphore. After that, it can safely work with the resource.

Upon completion of the work, the semaphore must be released so that other processes can access it. It goes without saying that all processes working with the shared resource should behave this way, i.e., access it through the semaphore[^13].

[^12]: So that various parts of the program have access to it.
[^13]: General rule: all processes working with a shared resource must behave this way, i.e., perform access through the semaphore.

These same considerations fully apply to calling a non-reentrant[^14] function.

[^14]: A function that uses objects with non-local storage class during its operation. Therefore, to prevent disruption of the program's integrity, such a function must not be called if an instance of the same function is already running.

!!! warning "**WARNING**"

    When using mutual exclusion semaphores, a situation may arise where one process, having locked a semaphore and working with the corresponding resource, attempts to access another resource, which is also accessed by locking another semaphore. This second semaphore is already locked by another process, which, in turn, is trying to access the resource already being used by the first process. As a result, both processes are waiting for each other to release the locked resource, and until that happens, neither can continue its work.

    This situation is called a "deadlock"[^15]. In English-language literature, it is denoted by the word "Deadlock." To avoid it, the programmer must carefully monitor access to shared resources. A good rule to prevent the situation described above is to lock no more than one mutual exclusion semaphore at a time. In any case, success here is based on the attentiveness and discipline of the program developer.

[^15]: Sometimes translated as "deadly embrace."

For implementing binary semaphores of this type, **scmRTOS** defines the `OS::TMutex` class. See "Listing 6. OS::TMutex".

```cpp
01    class TMutex : public TService
02    {
03    public:
04        INLINE TMutex() : ProcessMap(0), ValueTag(0) { }
05               void lock();
06               void unlock();
07               void unlock_isr();
08   
09        INLINE bool try_lock()        { TCritSect cs; if(ValueTag) return false;
10                                                      else lock(); return true; }
11        INLINE bool is_locked() const { TCritSect cs; return ValueTag != 0; }
12   
13    private:
14        volatile TProcessMap ProcessMap;
15        volatile TProcessMap ValueTag;
16   
17    };
```    
/// Caption
Listing 6. OS::TMutex
///

Obviously, before use, the semaphore must be created. Due to the specifics of its application, the semaphore should have the same storage class and scope as the resource it serves, i.e., it should be a static object with global visibility[^16].

[^16]: Although nothing prevents placing a Mutex outside the process code's scope and using a pointer or reference, either directly or through wrapper classes that automate the resource unlocking process via the automatic call of the wrapper class's destructor.

### Interface

---- 

#### <u>lock</u>

###### Prototype

```cpp
void TMutex::lock();
```

###### Description

Performs a blocking lock: if the semaphore was not previously locked by another process, its internal state is set to "locked," and the execution flow returns to the calling function. If the semaphore was already locked, the process will transition to a waiting state until the semaphore is released, and control is yielded to the kernel.

---- 

#### <u>unlock</u>

###### Prototype

```cpp
void TMutex::unlock();
```

###### Description

The function sets the internal state to "unlocked" and checks if any other process is waiting for this semaphore. If there is, control is yielded to the kernel, which will reschedule processes so that if the waiting process had higher priority, it will immediately gain control. If several processes were waiting for the semaphore, the highest-priority one among them will gain control. Only the process that locked the semaphore can unlock it—i.e., if this function is executed in a process that did not lock the mutex object, it will have no effect, and the object will remain in the same state.

---- 

#### <u>unlock from ISR</u>

###### Prototype

```cpp
INLINE void TMutex::unlock_isr();
```

###### Description

Sometimes a situation arises where a mutex is locked within a process, but work with the corresponding protected resource is performed in an interrupt handler (and this work is initiated in the process simultaneously with locking the mutex).

In this case, it is convenient to perform the unlock directly in the interrupt handler upon completion of work with the resource. For this purpose, `TMutex` includes the `unlock_isr()` function for unlocking the semaphore directly within an interrupt.

---- 

#### <u>try to lock</u>

###### Prototype

```cpp
INLINE bool TMutex::try_lock();
```

###### Description

Non-blocking lock. The difference from `lock()` is that the lock will only occur if the semaphore is free. For example, a process needs to work with a resource but also has a lot of other work. Attempting to lock with `lock()` might cause it to wait until the semaphore is freed, whereas that time could be spent on other work if available, and work with the shared resource could be performed only when access to it is not blocked.

This approach can be relevant for a high-priority process: if the semaphore is locked by a low-priority process, it is reasonable for the high-priority process not to yield control to the low-priority one if it has other work. Only when there is nothing left to do does it make sense to attempt to lock the semaphore in the usual way—by yielding control (since the low-priority process must also eventually gain control to finish its tasks and release the semaphore).

Considering the above, this function should be used with caution, as it could lead to a situation where a low-priority process never gains control because the high-priority one does not yield it.

---- 

#### <u>try to lock with timeout</u>

###### Prototype

```cpp
OS::TMutex::try_lock(timeout_t timeout);
```

###### Description

A blocking version of the previous function with a specified time interval—attempts to lock the semaphore within the specified timeout. If the lock is successful, returns `true`; otherwise, returns `false`.

---- 

#### <u>check if locked</u>

###### Prototype

```cpp
INLINE bool TMutex::is_locked()
```

###### Description

The function checks the state and returns `true` if the semaphore is locked, and `false` otherwise. It is sometimes convenient to use a semaphore as a status flag, where one process sets this flag (by locking the semaphore), and other processes check it and perform actions according to the state of that process.

---- 

### Usage Example

See "Listing 7. Example of Using OS::TMutex" for a usage example.

```cpp
01    OS::TMutex Mutex;
02    byte buf[16];
03    ...
04    template<> void TSlon::exec()
05    {
06        for(;;)
07        {
08            ...                           // some code
09                                          //
10            Mutex.lock();                 // resource access lock
11            for(byte i = 0; i < 16; i++)  //   
12            {                             //
13                ...                       // do something with buf
14            }                             //
15            Mutex.unlock();               // resource access unlock
16                                          //
17            ...                           // some code
18        }
19    }
```
/// Caption
Listing 7. Example of Using OS::TMutex
///

For convenience when using a mutual exclusion semaphore, the already mentioned wrapper class technique can be applied. In this case, it is implemented using the `TMutexLocker` class, included in the OS distribution. See "Listing 8. Wrapper Class OS::TMutexLocker".

```cpp
01     template <typename Mutex>
02     class TScopedLock
03     {
04     public:
05         TScopedLock(Mutex& m): mx(m) { mx.lock(); }
06         ~TScopedLock() { mx.unlock(); }
07     private:
08         Mutex & mx;
09     };
10 
11     typedef TScopedLock<OS::TMutex> TMutexLocker;
```
/// Caption
Listing 8. Wrapper Class OS::TMutexLocker
///

The methodology for using objects of this class is no different from using other wrapper classes—`TCritSect`, `TISRW`.

!!! tip "**ABOUT PRIORITY INVERSION**"

    A few words should be said about a mechanism related to mutual exclusion semaphores: priority inversion.

    The idea of priority inversion arises from the following situation. For example, a system has several processes, and processes with priorities N[^17] and N+n, where n>1, use the same resource, sharing work via a mutual exclusion semaphore. At some point, the process with priority N+n locks the semaphore and is working with the shared resource.

    During this, an event occurs that activates the process with priority N, which attempts to access the shared resource and, in trying to lock the semaphore, transitions to a waiting state. It will remain in this state until the process with priority N+n unlocks the semaphore. The delay of the higher-priority process in this situation is forced, as it is impossible to take control away from the process with priority N+n without violating the logic of its work and the integrity of access to the shared resource. Knowing this, the developer typically tries to minimize the time spent working with the resource so that the low-priority process does not block the work of the high-priority one.

    However, there is an unpleasant aspect to this situation: if at the moment described above a process with priority, for example, N+1, becomes active, it will preempt the process with priority N+n (since n>1) and thereby introduce an additional delay for the waiting higher-priority process with priority N. The situation is aggravated by the fact that the program developer usually does not connect the work of the process with priority N+1 with the manipulations of processes with priorities N and N+n on the shared resource. Therefore, they might not consider optimizing the work of the process with priority N+1 in relation to this, which could completely block the work of the process with priority N for an unpredictable amount of time. This is a highly undesirable situation.

    To avoid this, a technique called "priority inversion" is used. Its essence is that if a high-priority process attempts to lock a mutual exclusion semaphore that is already locked by a low-priority process, a temporary exchange of priorities occurs until the semaphore is unlocked. In effect, the low-priority process runs with the priority of the process that attempted to lock the semaphore. In this case, the situation described above, where a low-priority process blocks the work of a high-priority one, becomes impossible.

[^17]: It is assumed that process execution priority is inversely related to priority numbers—i.e., the process with priority 0 is the highest priority, and as priority numbers increase, process priority decreases.

Despite the coherence and elegance of the priority inversion method, it is not without drawbacks. The main one is that its technical implementation incurs overhead comparable to or exceeding the cost of implementing the mutual exclusion semaphore functionality itself. It might turn out that switching process priorities and everything associated with it—requiring consideration of all OS elements related to the priorities of processes involved in the inversion—slows down the system to an unacceptable level.

In connection with this, the priority inversion mechanism is not used in the current version of **scmRTOS**. Instead, to solve the aforementioned problem of a high-priority process being blocked by a low-priority one, a task delegation mechanism is proposed, discussed in detail in "Appendix A Usage Examples, A.1 Task Queue." This represents a unified method for redistributing the execution of contextually related code among processes with different priorities.

----

## OS::message

`OS::message` is a C++ template for creating objects that implement inter-process communication by transmitting structured data. `OS::message` is similar to `OS::TEventFlag` and differs mainly in that, besides the flag itself, it also contains an object of an arbitrary type constituting the actual message body.

The template definition is shown in "Listing 9 OS::message".

As can be seen from the listing, the message template is built upon the `TBaseMessage` class. This is done for efficiency reasons, to avoid duplicating common code in template instances—the code common to all messages is moved to the level of the base class[^18].

[^18]: The same technique is used when building the process template: the pair `class TBaseProcess`&nbsp;– `template process<>`.

```cpp
01    class TBaseMessage : public TService
02    {
03    public:
04        INLINE TBaseMessage() : ProcessMap(0), NonEmpty(false) { }
05   
06        bool wait  (timeout_t timeout = 0);
07        INLINE void send();
08        INLINE void send_isr();
09        INLINE bool is_non_empty() const { TCritSect cs; return NonEmpty;  }
10        INLINE void reset       ()       { TCritSect cs; NonEmpty = false; }
11   
12    private:
13        volatile TProcessMap ProcessMap;
14        volatile bool NonEmpty;
15    };
16   
17    template<typename T>
18    class message : public TBaseMessage
19    {
20    public:
21        INLINE message() : TBaseMessage()   { }
22        INLINE const T& operator= (const T& msg)
23        {
24            TCritSect cs;
25            *(const_cast<T*>(&Msg)) = msg; return const_cast<const T&>(Msg);
26        }
27        INLINE operator T() const
28        {
29            TCritSect cs;
30            return const_cast<const T&>(Msg);
31        }
32        INLINE void out(T& msg) { TCritSect cs; msg = const_cast<T&>(Msg); }
33   
34    private:
35        volatile T Msg;
36    };
```
/// Caption
Listing 9. OS::message
///

### Interface

----

#### <u>send</u>

###### Prototype

```cpp
INLINE void OS::TBaseMessage::send();
```

###### Description

Sends a message[^19]. The operation involves transitioning processes waiting for the message to the ready state and calling the scheduler.

---- 

#### <u>send from ISR</u>

###### Prototype

```cpp
INLINE void OS::TBaseMessage::send_isr();
```

###### Description

A variant of the above function optimized for use in interrupts. The function is inline and uses a special lightweight inline version of the scheduler. ***This variant must not be used outside of interrupt handler code***.

---- 

#### <u>wait</u>

###### Prototype

```cpp
bool OS::TBaseMessage::wait(timeout_t timeout);
```

###### Description

Waits for a message[^20]. The function checks if the message is non-empty. If it is non-empty, it returns `true`. If it is empty, it transitions the current process from the ready state to a state of waiting for this message.

If no argument is specified during the call or if the argument is 0, waiting will continue until some process sends a message or the current process is "awakened" using the `TBaseProcess::force_wake_up()` function[^21].

If an integer is specified as an argument, representing the timeout value expressed in system timer ticks, then waiting for the message will occur with a timeout, i.e., the process will be "awakened" in any case. If this happens before the timeout expires, meaning a message arrives before the timeout, the function returns `true`. Otherwise, i.e., if the timeout expires before the message is sent, the function returns `false`.

---- 

#### <u>check if non-empty</u>

###### Prototype

```cpp
INLINE bool OS::TBaseMessage::is_non_empty(); 
```

###### Description

The function returns `true` if a message has been sent, and `false` otherwise.

---- 

#### <u>reset</u>

###### Prototype

```cpp
INLINE OS::TBaseMessage::reset();
```

###### Description

The function resets the message, i.e., transitions it to the empty state. The message body remains unchanged.

---- 

#### <u>write message contents</u>

###### Prototype

```cpp
template<typename T>
INLINE const T& OS::message<T>::operator= (const T& msg);
```

###### Description

Writes the actual message contents into the message object. The standard way to use `OS::message` is to write the message body and send the message using the `TBaseMessage::send()` function—see "Listing 10. Using OS::message".

---- 

#### <u>access message body by reference</u>

###### Prototype

```cpp
template<typename T>
INLINE OS::message<T>::operator T() const;
```

###### Description

Returns a constant reference to the message body. This facility should be used with caution, bearing in mind that while accessing the message body via a reference, it might be modified in another process (or interrupt handler). Therefore, it is recommended to use the `message::out()` function for reading the message body.

---- 

#### <u>read message contents</u>

###### Prototype

```cpp
template<typename T>
INLINE OS::message<T>::out(T &msg);
```

###### Description

The function is intended for reading the message body. For efficiency, to avoid unnecessary copying of the message body, a reference to an external message body object is passed to the function. Inside the function, the message contents are copied into this external object.

---- 

[^19]: Analogous to the `OS::TEventFlag::signal()` function.
[^20]: Analogous to the `OS::TEventFlag::wait()` function.
[^21]: In the latter case, extreme caution is required.

### Usage Example

```cpp
01    struct TMamont { ... }           //  data type for sending by message
02    
03    OS::message<Mamont> mamont_msg;  // OS::message object
04    
05    template<> void Proc1::exec()
06    {
07        for(;;)
08        {
09            Mamont mamont;
10            mamont_msg.wait();      // wait for message
11            mamont_msg.out(mamont); // read message contents to the external object 
12            ...                     // using the Mamont contents
13        }     
14    }
15    
16    template<> void Proc2::exec()
17    {
18        for(;;)
19        {
20            ...
21            Mamont m;           // create message content
22    
23            m...  =             // message body filling
24            mamont_msg = m;     // put the content to the OS::message object
25            mamont_msg.send();  // send the message
26            ...
27        }
28    }
```
/// Caption
Listing 10. Using OS::message
///

----


## OS::channel

`OS::channel` is a C++ template for creating objects that implement ring buffers[^22] for the safe transmission of data of arbitrary types within a preemptive OS. Like any other interprocess communication service, `OS::channel` solves synchronization problems. The specific buffer type is specified during the template instantiation[^23] in the user code. The `OS::channel` template is based on the ring buffer template defined in the library included in the **scmRTOS** distribution:

```cpp
usr::ring_buffer<class T, uint16_t size, class S = uint8_t>`
```

[^22]: Functionally, it's a FIFO, i.e., a queue object for data transmission following the First In, First Out scheme.
[^23]: Creation of a class instance.

Building channels using C++ templates provides an efficient means for constructing message queues of arbitrary types. Unlike the dangerous, non-transparent, and inflexible method of organizing message queues based on `void *` pointers, the `OS::channel` queue offers:

*   **Safety through static type checking**, both when creating the queue/channel and when writing data to it or reading from it.
*   **Ease of use**—no need for manual type casting, which requires keeping extra information in mind about the real data types transmitted through the channel for their correct use.
*   **Significantly greater flexibility**—queue objects can be of any type, not just pointers.

Regarding the last point, a few words should be said: one drawback of using `void *` pointers as a basis for message passing is that the user must allocate memory for the messages themselves somewhere. This is extra work, and the target object becomes distributed—the queue is in one place, while the actual content of the queue elements is elsewhere.

The main advantages of the pointer-based message mechanism are high efficiency for large message bodies and the ability to transmit messages of varying formats. However, if, for example, messages are small—a few bytes—and all have the same format, there is no need for pointers. It's much simpler to create a queue of the required number of such messages, and that's it. In this case, as mentioned, there's no need to allocate memory for the message bodies—since messages are placed entirely into the channel queue, memory for them will be automatically allocated by the compiler directly when creating the channel.

The channel template definition is shown in "Listing 11. Definition of the OS::channel Template".

```cpp
01    template<typename T, uint16_t Size, typename S = uint8_t>
02    class channel : public TService
03    {
04    public:
05        INLINE channel() : ProducersProcessMap(0)
06                         , ConsumersProcessMap(0)
07                         , pool()
08        {
09        }
10
11        //----------------------------------------------------------------
12        //
13        //    Data transfer functions
14        //
15        void write(const T* data, const S cnt);
16        bool read (T* const data, const S cnt, timeout_t timeout = 0);
17
18        void push      (const T& item);
19        void push_front(const T& item);
20
21        bool pop     (T& item, timeout_t timeout = 0);
22        bool pop_back(T& item, timeout_t timeout = 0);
23
24        //----------------------------------------------------------------
25        //
26        //    Service functions
27        //
28        INLINE S get_count()     const;
29        INLINE S get_free_size() const;
30        void flush();
31
32    private:
33        volatile TProcessMap ProducersProcessMap;
34        volatile TProcessMap ConsumersProcessMap;
35        usr::ring_buffer<T, Size, S> pool;
36    };
```
/// Caption
Listing 11. Definition of the OS::channel Template
///

`OS::channel` is used as follows: first, define the type of objects to be transmitted through the channel, then define the channel type or create a channel object. For example, suppose the data transmitted through the channel is a structure:

```cpp
struct Data
{
    int   a;
    char *p;
};
```

Now, you can create a channel object by instantiating the `OS::channel` template:

```cpp
OS::channel<Data, 8> data_queue;
```

This code declares a channel object `data_queue` for transmitting objects of type `Data`, with a channel capacity of 8 objects. The channel is now ready for use.

`OS::channel` provides the ability to write data not only to the end of the queue but also to the beginning, and to read data not only from the beginning but also from the end of the queue. When reading, it is also possible to specify a timeout value.

The following interface is provided for operating on a channel object:

### Interface

----

#### <u>push</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
void OS::channel<T, Size, S>::push(const T& item);
```

###### Description

Pushes one element to the end of the queue[^24]. If there was space in the channel at the time of writing, the element is written to the queue and the scheduler is called. If there was no space, the process transitions to a waiting state until space becomes available in the channel. When space appears, the element will be written, followed by a call to the scheduler.

----

#### <u>push front</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
void OS::channel<T, Size, S>::push_front(const T& item);
```

###### Description

Pushes an element to the **beginning** of the queue. In all other respects, the logic of operation is exactly the same as for `channel::push()`.

----

#### <u>pop</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
bool OS::channel<T, Size, S>::pop(T& item, timeout_t timeout);
```

###### Description

Pops one element from the **beginning** of the queue if the channel was not empty. If the channel was empty, the process transitions to a waiting state until data appears in it **or** until the timeout expires, if a timeout was specified[^25].

In case of a call with a timeout: if data arrives before the timeout expires, the function returns `true`; otherwise, it returns `false`. If the call was made without a timeout, the function always returns `true`, except when awakened by `OS::TBaseProcess::wake_up()` or `OS::TBaseProcess::force_wake_up()`.

In all cases, when an element is popped, the scheduler is called.

Note that when calling this function, the data popped from the channel is not returned by copying from the function but is passed via a reference to an object. This is because the return value is used to convey the timeout result.

----

#### <u>pop back</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
bool OS::channel<T, Size, S>::pop_back(T& item, timeout_t timeout);
```

###### Description

Pops one element from the **end** of the queue if the channel was not empty. All functionality is exactly the same as for `channel::pop()`.

----

#### <u>write</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
void OS::channel<T, Size, S>::write(const T* data, const S count);
```

###### Description

Writes several elements from memory at the given address to the **end** of the queue. Essentially, this is the same as pushing one element to the end of the queue (`push`), except that not one but the specified number of elements are written. In case of waiting, it continues until enough space becomes available in the channel.

----

#### <u>write inside ISR</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::write_isr(const T* data, const S count);
```

###### Description

A special version for use inside interrupt handlers. The function writes as many elements as the free space in the channel allows (but no more than the specified count) and returns the number of elements written. Processes waiting for data from the channel are transitioned to the ready state.

The call is **non-blocking**. The scheduler is **not** called.

----

#### <u>read</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
bool OS::channel<T, Size, S>::read(T* const data, const S count, timeout_t timeout);
```

###### Description

Pops several elements from the channel. This is the same as `channel::pop()`, except that not one but the specified number of elements are popped. If waiting occurs, it continues until the required number of elements is available in the channel or until the timeout triggers (if used).

----

#### <u>read inside ISR</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::read_isr(T* const data, const S max_size);
```

###### Description

A special version for use inside interrupt handlers. The function reads as many elements as are present in the channel (but no more than the specified count) and returns the number of elements written. Processes waiting for free space to appear in the channel are transitioned to the ready state.

The call is **non-blocking**. The scheduler is **not** called.

----

#### <u>get item count</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::get_count();
```

###### Description

Returns the number of elements currently in the channel. The function is **inline**, so its efficiency is maximized.

----

#### <u>get free size</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::get_free_size();
```

###### Description

Returns the amount of free space available in the channel.

----

#### <u>flush</u>

###### Prototype

```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::flush();
```

###### Description

Clears the channel. The function clears the buffer by calling `usr::ring_buffer<>::flush()` and calls the scheduler.

----

[^24]: Refers to the channel's queue. Functionally, since a channel is a FIFO, the end of the queue corresponds to the FIFO input, and the beginning of the channel corresponds to the FIFO output.
[^25]: I.e., the call included a second argument—an integer specifying the timeout value in system timer ticks.

### Usage Example

A simple usage example is shown in "Listing 12. Example of Using a Queue Based on a Channel".

```cpp
01    //---------------------------------------------------------------------
02    struct Cmd
03    {
04        enum CmdName { cmdSetCoeff1, cmdSetCoeff2, cmdCheck } CmdName;
05        int Value;
06    };
07
08    OS::channel<Cmd, 10> cmd_q; // Queue for Commands with 10 items depth
09    //---------------------------------------------------------------------
10    template<> void Proc1::exec()
11    {
12        ...
13        Cmd cmd = { cmdSetCoeff2, 12 };
14        cmd_q.push(cmd);
15        ...
16    }
17    //---------------------------------------------------------------------
18    template<> void Proc2::exec()
19    {
20        ...
21        Cmd cmd;
22        if( cmd_q.pop(cmd, 10) ) // wait for data, timeout 10 system ticks
23        {
24            ... // data incoming, do something
25        }
26        else
27        {
28            ... // timeout expires, do something else
29        }
30        ...
31    }
32    //---------------------------------------------------------------------
```
/// Caption
Listing 12. Example of Using a Queue Based on a Channel.
///

As can be seen, usage is quite simple and transparent. In one process (Proc1), a command message `cmd` is created (13), initialized with the required values, and written to the channel queue (14). In another process (Proc2), there is a wait for data from the queue (22). If data arrives, the corresponding code is executed (23)-(25); if the timeout expires, different code is executed (27)-(29).

----

## Final Remarks

There is a certain invariant relationship among different interprocess communication services. That is, using some services (or, more often, a combination thereof), you can accomplish the same task as with others. For example, instead of using a channel, you could create a static array and exchange data through it using mutual exclusion semaphores to prevent concurrent access and event flags to notify the waiting process that data is ready for it. In some cases, such an implementation might be more efficient, albeit less convenient.

You could use messages for event synchronization instead of event flags—this approach makes sense if you also need to transmit some information along with the flag. In fact, that's precisely what `OS::message` is designed for. The variety of uses is great, and which option is best suited for a particular situation is determined primarily by the situation itself.

!!! info "**TIP**"

    It is necessary to understand and remember that any interprocess communication service, while performing its functions, does so within a critical section, i.e., with interrupts disabled. Therefore, you should not overuse interprocess communication services where they can be avoided.

    For example, using a mutual exclusion semaphore when accessing a static variable of a built-in type is not a good idea compared to simply using a critical section, because the semaphore also uses critical sections when locking and unlocking, and the time spent in them is longer than during simple variable access.

There are certain peculiarities when using services within interrupts. For example, it is obvious that using `TMutex::lock()` inside an interrupt handler is a rather bad idea because, firstly, mutual exclusion semaphores are intended for resource sharing at the process level, not the interrupt level. Secondly, waiting for a resource to be released inside an interrupt handler will not work anyway and will only lead to the process interrupted by this interrupt being transitioned to a waiting state at an inappropriate and unpredictable point. Effectively, the process will be placed in an inactive state, from which it can only be recovered using the `TBaseProcess::force_wake_up()` function. In any case, nothing good will come of this.

A somewhat similar situation can arise when using channel objects in an interrupt handler. You cannot wait for data from a channel inside an ISR, and the consequences will be similar to those described above. Writing data to a channel is also not entirely safe. If, for instance, there is insufficient space in the channel when writing, the program's behavior will be far from what the user expects.

!!! warning "**RECOMMENDATION**"

    For work inside interrupts, you should use the service member functions with the `_isr` suffix—these are specially developed versions that ensure efficiency and safe use of interprocess communication services within interrupts.

And, of course, if the existing set of interprocess communication services does not meet the needs of a particular project for some reason, it is always possible to design a custom service class based on the provided foundation in the form of `TService`. In doing so, the standard set of services can serve as design examples.
