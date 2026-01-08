# Interprocess<br>Communication<br>Services

## Introduction

Starting with version 4, **scmRTOS** employs a fundamentally different mechanism for implementing interprocess communication services compared to previous versions. Previously, each service class was developed individually with no relation to the others, and service classes were declared as "friends" of the kernel to access its resources. This approach prevented code reuse[^1] and made it impossible to extend the set of services, leading to its abandonment in favor of a design free from both drawbacks.

[^1]: Since interprocess communication services perform similar operations when interacting with kernel resources, they contained nearly identical code in several places.

The implementation is based on the concept of extending OS functionality through extension classes derived from `TKernelAgent` (see ["TKernelAgent and Extensions"](kernel.md#kernel-agent)).

The key class for building interprocess communication services is `TService`, which implements the common functionality shared by all service classes. All service classes—both those included in the standard **scmRTOS** distribution and those developed[^2] as extensions to the standard set—are derived from `TService`.

[^2]: Or that can be developed by the user to meet the needs of their project.

The interprocess communication services included in **scmRTOS** distribution are:

* `OS::TEventFlag`;
* `OS::TMutex`;
* `OS::message`;
* `OS::channel`;

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

Like its parent class `TKernelAgent`, the `TService` class does not allow instantiation of objects of its own type: its purpose is to provide a base for constructing concrete types&nbsp;– interprocess communication services. The interface of this class consists of a set of functions that express the common actions performed by any service class in the context of control transfer between processes. Logically, these functions can be divided into two groups: core and auxiliary.

The auxiliary functions include:

1. `TService::cur_proc_prio_tag()`. Returns the tag[^3] corresponding to the currently active process. This tag is actively used by the core service functions to record process identifiers[^4] when placing the current process into a waiting state.
2. `TService::highest_prio_tag()`. Returns the tag of the highest-priority process from the process map passed as an argument. It is primarily used to obtain the identifier (of the process) from those recorded in the service object's process map, identifying the process that should be marked ready to run.

[^3]: A process tag is technically a mask of type `TProcessMap` with only one non-zero bit. The position of this bit in the mask corresponds to the process priority. Process tags are used to manipulate `TProcessMap` objects, which represent process readiness/unreadiness for execution, as well as to record process tags.
[^4]: Alongside the process priority number, the tag can also serve as a process identifier&nbsp;– there is a one-to-one correspondence between a process priority and its tag. Each identifier type has efficiency advantages in specific situations, so both are extensively used in the OS code.

The core functions are:

1. `TService::suspend()`. Places the process into an unready state, records the process identifier in the service's process map, and invokes the OS scheduler. This function forms the basis for service member functions used to wait for an event (`wait()`, `pop()`, `read()`) or for actions that may involve waiting for resource release (`lock()`, `push()`, `write()`).
2. `TService::is_timeouted()`. Returns `false` if the process was marked ready-to-run via a service member function call; returns `true` if the process was marked ready-to-run due to timeout expiration[^5] or forcibly via `TBaseProcess` member functions `wake_up()` or `force_wake_up()`. The result is used to determine whether the process successfully waited for the expected event (or resource release) or not.
3. `TService::resume_all()`. Checks for processes recorded in the service's process map that are in an unready state[^6]; if any exist, all are marked ready and the scheduler is invoked. The function returns `true` in this case, otherwise `false`.
4. `TService::resume_next_ready()`. Performs actions similar to `resume_all()`, but with the difference that, when waiting processes are present, only one—the highest-priority—is marked ready instead of all.

[^5]: In other words, "awakened" in the system timer handler.
[^6]: I.e., processes whose waiting state was not interrupted by timeout and/or forcibly via `TBaseProcess::wake_up()` or `TBaseProcess::force_wake_up()`.

For the `resume_all()` and `resume_next_ready()` functions, there are versions optimized for use inside interrupt handlers: `resume_all_isr()` and `resume_next_ready_isr()`. In purpose and semantics, they resemble the main variants[^7]; the primary difference is that they do not invoke the scheduler.

[^7]: Therefore, a full description is not provided.

### Usage

#### Preliminary Notes

Any service class is created by inheriting from `TService`. As an example of using `TService` and building a service class upon it, one of the standard interprocess communication services—`TEventFlag`—will be examined:

```cpp
class TEventFlag : public TService { ... }
```

The `TEventFlag` service class itself will be described in detail later; for narrative continuity, it should be noted here that this interprocess communication service is used to synchronize process operation with occurring events. The basic usage idea: one process waits for an event using the `TEventFlag::wait()` member function, while another process[^8] signals the flag using `TEventFlag::signal()` when the event that needs handling in the waiting process occurs.

[^8]: Or an interrupt handler&nbsp;– depending on the event source. A special version of the signaling function exists for interrupt handlers, but in the current context this detail is immaterial and therefore omitted.

Given the above, primary attention in this example will focus on these two functions, as they carry the main semantic load of the service class[^9] and its development largely reduces to implementing such functions.

[^9]: The rest of its interface is auxiliary, serving to complete the class and improve its usability.

#### Requirements for the Developed Class Functions

Requirements for the event flag wait function: the function must check whether the event has already occurred at the moment of call and, if not, be capable of waiting[^10] for the event either unconditionally or with a timeout condition. Return `true` if exiting due to the event; return `false` if exiting due to timeout.

[^10]: I.e., yield control to the kernel and remain in passive waiting.

Requirements for the event flag signal function: the function must mark all processes waiting for the event flag as ready-to-run and transfer control to the scheduler.

#### Implementation

Inside the `wait()` member function—see "Listing 2. TEventFlag::wait()"—the first step is to check whether the event has already been signaled; if so, the function returns `true`. If the event has not been signaled (i.e., it needs to be awaited), preparatory actions are performed: in particular, the wait timeout value is written to the current process's `Timeout` variable, and the `suspend()` function defined in the base class `TService` is called. This function records the current process tag in the event flag object's process map (passed as an argument to `suspend()`), marks the process unready, and yields control to other processes by invoking the scheduler.

Upon return from `suspend()`—meaning the process has been marked ready—a check determines the source of the awakening. This is done by calling `is_timeouted()`, which returns `false` if the process was awakened via `TEventFlag::signal()` (i.e., the expected event occurred without timeout) and `true` if awakening occurred due to the timeout specified in the `TEventFlag::wait()` argument or forcibly.

This logic for the `TEventFlag::wait()` member function enables efficient use in user code when organizing process operation synchronized with required events[^11], while keeping the implementation code simple and transparent.

[^11]: Including cases where the events do not occur within the specified time interval.

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
Listing 2. The TEventFlag::wait() Function  
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
Listing 3. The TEventFlag::signal() Function  
///

The code for `TEventFlag::signal()`—see "Listing 3. TEventFlag::signal()"—is extremely simple: it marks all processes waiting for this event flag as ready-to-run and performs rescheduling. If none were waiting, the internal event flag variable `efOn` is set to `true`, indicating that the event occurred but has not yet been handled.

Any interprocess communication service can be designed and implemented in a similar manner. During development, it is only necessary to clearly understand what the `TService` member functions do and use them appropriately.

----

## OS::TEventFlag

In program execution, it is often necessary to synchronize processes. For example, one process must wait for an event before continuing its work. This can be handled in various ways: the process might continuously poll a global flag in a tight loop, or it could poll periodically: check the flag, sleep with a timeout, wake up, check again, and so on. The first approach is poor because the polling process, due to its tight loop, prevents lower-priority processes from receiving any CPU time: they cannot preempt the polling process despite their lower priorities.

The second approach is also suboptimal: the polling period is relatively large (resulting in low temporal resolution), and during each poll the process consumes CPU cycles solely for context switching, even though it is unknown whether the event has occurred.

A proper solution is to place the process into a waiting state for the event and transfer control to it only when the event actually occurs.

This functionality in **scmRTOS** is provided by `OS::TEventFlag` objects (event flags). The class definition is shown in "Listing 4. OS::TEventFlag".

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
Wait for an event. When `wait()` is called, the following occurs: the flag is checked to see if it is set. If it is, the flag is cleared and the function returns `true`, meaning the event had already occurred at the time of the call. If the flag is not set (i.e., the event has not yet occurred), the process is placed into a waiting state for this flag (event), and control is yielded to the kernel, which reschedules processes and runs the next ready-to-run one.

If the function is called without an argument (or with an argument of 0), the process will remain waiting until the event flag is signaled by another process (using `signal()`) or an interrupt handler (using `signal_isr()`) or until it is forcibly awakened using `TBaseProcess::force_wake_up()` (the latter should be used with extreme caution).

When `wait()` is called without an argument, it always returns `true`. If called with a positive integer argument representing a timeout in system timer ticks, the process waits as described, but if the event flag is not signaled within the specified period, the process is awakened by the timer and the function returns `false`. This implements both unconditional waiting and waiting with timeout.

----
#### <u>signal</u>
###### Prototype
```cpp
INLINE void OS::TEventFlag::signal();
```

###### Description
A process that wishes to notify other processes via a `TEventFlag` object that a particular event has occurred must call `signal()`. This marks all processes waiting for the event as ready to run, and control is immediately transferred to the highest-priority one among them (the others will run in priority order).

----
#### <u>signal from ISR</u>
###### Prototype
```cpp
INLINE void OS::TEventFlag::signal_isr();
```

###### Description
A version of the above function optimized for use in interrupt service routines. The function is inline and uses a special lightweight inline version of the scheduler. This variant must not be used outside interrupt handler code.

----
#### <u>clear</u>
###### Prototype
```cpp
INLINE void OS::TEventFlag::clear();
```

###### Description
Clear the flag. Sometimes synchronization requires waiting for the *next* event rather than processing one that has already occurred. In such cases, the event flag must be cleared before starting the wait. The `clear()` function serves this purpose.

----
#### <u>is_signaled</u>
###### Prototype
```cpp
INLINE bool OS::TEventFlag::is_signaled();
```

###### Description
Check the flag state. It is not always necessary to wait for an event by yielding control. Sometimes the program logic only requires checking whether the event has occurred.

----
### Usage Example

An example of using an event flag is shown in "Listing 5. Using TEventFlag".

In this example, process `Proc1` waits for an event with a timeout of 10 system timer ticks (line 9). The second process—`Proc2`—signals the flag when a condition is met (line 27). If the first process has higher priority, it will immediately receive control.

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
    When an event occurs and a process signals the flag, **all** processes waiting for that flag are marked ready-to-run. In other words, every process that was waiting will be awakened. They will, of course, receive control in order of their priorities, but no process that had already entered the wait state will miss the event, regardless of its priority.

    Thus, the event flag **possesses a broadcast property**, which is very useful for notifying and synchronizing multiple processes on a single event. Naturally, nothing prevents using an event flag in a point-to-point manner, where only one process is waiting for the event.

----

## OS::TMutex

A Mutex semaphore (from *Mutual Exclusion*) is designed, as its name suggests, to enforce mutual exclusion in access. At any given moment, only one process may hold (own) the mutex. If another process attempts to acquire a mutex that is already held by a different process, the attempting process will wait until the mutex is released.

The primary use of mutex semaphores is to provide mutual exclusion when accessing shared resources. For example, consider a static array with global scope[^12] through which two processes exchange data. To prevent errors during the exchange, access to the array must be exclusive&nbsp;– one process must not be allowed to access it while the other is working with it.

Using a critical section for this purpose is not ideal, because interrupts would be disabled for the entire duration of the array access, which can be significant. During this time, the system would be unable to respond to events. A mutual-exclusion semaphore is far better suited here: the process intending to work with the shared resource must first acquire the mutex. Once acquired, it can safely manipulate the resource.

Upon completion, the process must release the mutex so that other processes can gain access. It goes without saying that all processes accessing the shared resource must follow this discipline: accessing it only through the mutex[^13].

[^12]: To make it accessible to different parts of the program.
[^13]: General rule: every process working with a shared resource must adhere to this protocol.

The same considerations fully apply to calling non-reentrant[^14] functions.

[^14]: A function that uses objects with non-local storage duration during its execution; calling it while another instance is already running would corrupt program integrity.

!!! warning "**WARNING**"
    When using mutual-exclusion semaphores, a deadlock situation (sometimes mentioned as "deadly embrace") can arise. Imagine one process holds Mutex A and attempts to acquire Mutex B, while another process holds Mutex B and attempts to acquire Mutex A. Both processes end up waiting indefinitely for the other to release its mutex.

    This is known as a *deadlock*. To avoid it, the programmer must carefully manage access to shared resources. A good rule that prevents such situations is to **never hold more than one mutex at a time**. In any case, success depends on the developer's attention and discipline.

Binary semaphores of this type are implemented in **scmRTOS** by the class `OS::TMutex`, see "Listing 6. OS::TMutex".

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

Obviously, a mutex must be created before use. Due to its purpose, the mutex should have the same storage class and visibility as the resource it protects&nbsp;– typically a static object with global scope[^15].

[^15]: Although nothing prevents placing the mutex outside a process's visibility and accessing it via pointer/reference, either directly or through wrapper classes that automate unlocking via their destructor.

### Interface
----

#### <u>lock</u>
###### Prototype
```cpp
void TMutex::lock();
```

###### Description
Performs a blocking acquire: if the mutex is currently free, its internal state is set to locked and control returns to the caller. If the mutex is already held, the calling process is placed in a waiting state until the mutex is released, and control is yielded to the kernel.

----
#### <u>unlock</u>
###### Prototype
```cpp
void TMutex::unlock();
```

###### Description
Sets the internal state to unlocked and checks whether any other process is waiting for this mutex. If so, control is yielded to the kernel for rescheduling&nbsp;– the highest-priority waiting process will receive control immediately if it has higher priority. If multiple processes are waiting, the highest-priority one runs next. **Only the process that locked the mutex can unlock it!** Calling `unlock()` from a different process has no effect and leaves the mutex locked.

----
#### <u>unlock from ISR</u>
###### Prototype
```cpp
INLINE void TMutex::unlock_isr();
```

###### Description
Sometimes a mutex is locked in a process, but the actual work with the protected resource occurs in an interrupt handler (initiated by the process after locking). In such cases, it is convenient to unlock mutex directly from the ISR upon completion. The `unlock_isr()` function is provided for this purpose.

----
#### <u>try to lock</u>
###### Prototype
```cpp
INLINE bool TMutex::try_lock();
```

###### Description
Non-blocking acquire. Unlike `lock()`, acquisition occurs only if the mutex is currently free. This is useful when a process has other work to do and does not want to block indefinitely. For example, a high-priority process might prefer to perform alternative tasks while a lower-priority process holds the mutex, rather than yielding control.

Use this function cautiously: excessive use in high-priority processes can starve lower-priority ones, preventing them from ever releasing the mutex.

----
#### <u>try to lock with timeout</u>
###### Prototype
```cpp
OS::TMutex::try_lock(timeout_t timeout);
```

###### Description
Blocking acquire limited to the specified timeout interval. Returns `true` if the mutex was acquired, `false` otherwise.

----
#### <u>check if locked</u>
###### Prototype
```cpp
INLINE bool TMutex::is_locked()
```

###### Description
Returns `true` if the mutex is currently locked, `false` otherwise. Sometimes a mutex is used as a simple state flag&nbsp;– one process sets it (by locking), while others check the state and react accordingly.

----
### Usage Example

An example is shown in "Listing 7. Example of Using OS::TMutex".

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

For convenient mutex usage, the well-known wrapper-class technique can be applied via `TMutexLocker` (included in the distribution), see "Listing 8. Wrapper Class OS::TMutexLocker".

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

The usage methodology is identical to other wrappers such as `TCritSect` and `TISRW`.

<a name="mutex-priority-inversion"></a>
!!! tip "**ON PRIORITY INVERSION**"
    A few words about priority inversion, a phenomenon related to mutual-exclusion semaphores.

    Consider a system with processes of priorities N[^17] and N+n (n>1) sharing a resource protected by a mutex. The higher-priority process (N) attempts to acquire the mutex while the lower-priority process (N+n) already holds it and is working with the resource. The high-priority process must wait&nbsp;– an unavoidable delay, as preempting the low-priority process would corrupt resource integrity. Developers typically minimize the critical section[^16] duration to limit this delay.

    The problem arises when a medium-priority process (e.g., N+1) becomes ready: it preempts the low-priority holder (N+n), further delaying the high-priority waiter (N). Since the medium-priority process is unrelated to the shared resource, its execution time may not be optimized, potentially blocking the high-priority process indefinitely causes an undesirable situation.


    To prevent this, priority inheritance is used is used: when a high-priority process waits on a mutex held by a low-priority process, the holder temporarily inherits the waiter’s priority until it releases the mutex. This eliminates unbounded blocking.

[^16]: Critical section in this context means time-critical access, not the OS critical section object.

[^17]: In this example, priorities are inversely related to their numeric value&nbsp;– priority 0 is highest; higher numbers mean lower priority.

Despite its elegance, priority inheritance has drawbacks: implementation overhead can be comparable to or exceed that of the mutex itself, and the required modifications across the OS (all priority-related components) may unacceptably degrade performance.

For these reasons, the current version of **scmRTOS** does not implement priority inheritance. Instead, the problem is addressed via task delegation, described in detail in Appendix ([Example: Job Queue](example-job-queue.md)), which provides a unified method for redistributing context-related code execution across processes of different priorities.

----

## OS::message

`OS::message` is a C++ template for creating objects that enable interprocess communication by exchanging structured data. `OS::message` is similar to `OS::TEventFlag`, with the main difference being that, in addition to the flag itself, it also contains an object of an arbitrary type that forms the actual message payload.

The template definition is shown in "Listing 9. OS::message".

As can be seen from the listing, the message template is built upon the `TBaseMessage` class. This is done for efficiency reasons&nbsp;– to avoid duplicating common code across template instantiations. The code shared by all messages is factored out into the base class[^18].

[^18]: The same technique is used in the process implementation: the pair `class TBaseProcess`&nbsp;– `template process<>`.

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
Send the message[^19]: the operation marks all processes waiting for the message as ready to run and invokes the scheduler.

----
#### <u>send from ISR</u>
###### Prototype
```cpp
INLINE void OS::TBaseMessage::send_isr();
```

###### Description
A version of the above function optimized for use in interrupt handlers. The function is inline and uses a special lightweight inline scheduler version. *This variant must not be used outside interrupt handler code*.

----
#### <u>wait</u>
###### Prototype
```cpp
bool OS::TBaseMessage::wait(timeout_t timeout);
```

###### Description
Wait for a message[^20]: the function checks whether the message is non-empty. If it is, the function returns `true`. If it is empty, the current process is removed from the ready-to-run state and placed into a waiting state for this message.

If called without an argument (or with an argument of 0), waiting continues indefinitely until another process sends a message or the current process is forcibly awakened using `TBaseProcess::force_wake_up()`[^21].

If a positive integer timeout (in system timer ticks) is provided, waiting occurs with a timeout&nbsp;– the process will be awakened in any case. If awakened before the timeout expires (i.e., a message arrives), the function returns `true`. If the timeout expires first, the function returns `false`.

----
#### <u>check if non-empty</u>
###### Prototype
```cpp
INLINE bool OS::TBaseMessage::is_non_empty();
```

###### Description
Returns `true` if a message has been sent (non-empty), `false` otherwise.

----
#### <u>reset</u>
###### Prototype
```cpp
INLINE OS::TBaseMessage::reset();
```

###### Description
Resets the message to the empty state. The message payload remains unchanged.

----
#### <u>write message contents</u>
###### Prototype
```cpp
template<typename T>
INLINE const T& OS::message<T>::operator= (const T& msg);
```

###### Description
Writes the message payload into the message object. The standard way to use `OS::message` is to write the payload and then send the message using `TBaseMessage::send()`, see "Listing 10. Using OS::message".

----
#### <u>access message body by reference</u>
###### Prototype
```cpp
template<typename T>
INLINE OS::message<T>::operator T() const;
```

###### Description
Returns a constant reference to the message payload. Use this cautiously: while accessing the payload via reference, it may be modified by another process (or interrupt handler). For safe reading, prefer the `message::out()` function.

----
#### <u>read message contents</u>
###### Prototype
```cpp
template<typename T>
INLINE OS::message<T>::out(T &msg);
```

###### Description
Intended for reading the message payload. To avoid unnecessary copying, a reference to an external payload object is passed; the function copies the message contents into it.

----
[^19]: Analogous to `OS::TEventFlag::signal()`.
[^20]: Analogous to `OS::TEventFlag::wait()`.
[^21]: The latter should be used with extreme caution.

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

`OS::channel` is a C++ template for creating objects that implement ring buffers[^22] for safe, preemption-aware data transfer of arbitrary types in a preemptive RTOS. Like any other interprocess communication service, `OS::channel` also handles synchronization.

The specific ring buffer type is defined at template instantiation[^23] in user code. The `OS::channel` template is built upon a generic ring buffer template provided in the **scmRTOS** distribution library:

```cpp
usr::ring_buffer<class T, uint16_t size, class S = uint8_t>
```

[^22]: Functionally, this is a FIFO (First In – First Out) queue for data transfer.
[^23]: Instantiation of the template class.

Building channels from C++ templates provides an efficient way to create message queues of arbitrary types. Compared to the unsafe, opaque, and inflexible approach of using `void *` pointers for message queues, `OS::channel` offers:

* Type safety through static type checking&nbsp;– both when creating the channel and when writing/reading data.
* Ease of use&nbsp;– no manual type casts are required, eliminating the need to keep track of actual data types for correct usage.
* Greater flexibility&nbsp;– queue elements can be any type, not just pointers.

Regarding the last point: the drawback of `void *`-based message passing is that the user must allocate memory for the messages themselves. This adds extra work and results in distributed objects: the queue in one place, the message payloads elsewhere.

The main advantages of pointer-based messages are high efficiency with large payloads and the ability to transfer heterogeneous messages. However, when messages are small (a few bytes) and uniformly formatted, pointers are unnecessary. It is far simpler to create a queue holding the required number of such messages directly. As noted, no separate memory allocation is needed for payloads&nbsp;– the compiler automatically allocates storage for the entire message within the channel upon creation.

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

`OS::channel` is used as follows: first define the type of objects to be transferred, then either define the channel type or create a channel object. For example, suppose the data to be transferred is a structure:

```cpp
struct Data
{
    int   a;
    char *p;
};
```

A channel object can now be created by instantiating the `OS::channel` template:

```cpp
OS::channel<Data, 8> data_queue;
```

This declares a channel object `data_queue` for transferring `Data` objects, with a capacity of 8 items. The channel is now ready for data transfer.

`OS::channel` supports writing data not only to the tail but also to the head of the queue, and reading not only from the head but also from the tail. Reading operations allow specifying a timeout.

The following interface is provided for channel operations:

### Interface
----

#### <u>push</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
void OS::channel<T, Size, S>::push(const T &item);
```

###### Description
Writes a single element to the tail of the queue[^24]. If space is available, the element is written and the scheduler is invoked. If no space is available, the process waits until space appears, then writes the element and invokes the scheduler.

----
#### <u>push_front</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
void OS::channel<T, Size, S>::push_front(const T &item);
```

###### Description
Writes an element to the head of the queue; otherwise identical to `channel::push()`.

----
#### <u>pop</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
bool OS::channel<T, Size, S>::pop(T &item, timeout_t timeout);
```

###### Description
Extracts a single element from the head of the queue if the channel is not empty. If empty, the process waits until data arrives or the timeout expires (if specified)[^25].

When called with a timeout, returns `true` if data arrived before timeout expiration, `false` otherwise. When called without timeout, always returns `true` (except when awakened by `OS::TBaseProcess::wake_up()` or `OS::TBaseProcess::force_wake_up()`).

In all cases, extracting an element invokes the scheduler.

Note that extracted data is returned via reference parameter rather than function return value, as the return value is used for timeout status.

----
#### <u>pop_back</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
bool OS::channel<T, Size, S>::pop_back(T &item, timeout_t timeout);
```

###### Description
Extracts a single element from the tail of the queue; otherwise identical to `channel::pop()`.

----
#### <u>write</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
void OS::channel<T, Size, S>::write(const T *data, const S count);
```

###### Description
Writes multiple elements to the tail from a memory buffer. Equivalent to repeated `push()`, but waits (if necessary) until sufficient space is available for the entire block.

----
#### <u>write inside ISR</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::write_isr(const T *data, const S count);
```

###### Description
Special version for use inside interrupt handlers. Writes as many elements as free space allows (up to the requested count) and returns the number written. Waiting consumers are marked ready.

Non-blocking. Scheduler is not invoked.

----
#### <u>read</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
bool OS::channel<T, Size, S>::read(T *const data, const S count, timeout_t timeout);
```

###### Description
Extracts multiple elements from the channel. Equivalent to repeated `pop()`, but waits (if necessary) until the requested number of elements is available or timeout expires.

----
#### <u>read inside ISR</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::read_isr(T *const data, const S max_size);
```

###### Description
Special version for interrupt handlers. Reads as many elements as available (up to the requested maximum) and returns the number read. Waiting producers are marked ready.

Non-blocking. Scheduler is not invoked.

----
#### <u>get item count</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::get_count();
```

###### Description
Returns the current number of elements in the channel. Inline for maximum efficiency.

----
#### <u>get free size</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::get_free_size();
```

###### Description
Returns the amount of free space in the channel.

----
#### <u>flush</u>
###### Prototype
```cpp
template<typename T, uint16_t Size, typename S>
S OS::channel<T, Size, S>::flush();
```

###### Description
Clears the channel by calling `usr::ring_buffer<>::flush()` and invokes the scheduler.

----
[^24]: Referring to the channel queue. Since the channel is a FIFO, the tail corresponds to the FIFO input, the head to the FIFO output.
[^25]: I.e., the call included a second argument specifying the timeout in system timer ticks.

### Usage Example

A simple example is shown in "Listing 12. Example of Using a Queue Based on a Channel".

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
Listing 12. Example of Using a Queue Based on a Channel  
///

As shown, usage is straightforward and clear. In one process (`Proc1`), a command message `cmd` is created (line 13), initialized, and written to the channel queue (line 14). In the other process (`Proc2`), data is awaited from the queue (line 22); upon arrival, corresponding code executes (lines 23–25), while timeout triggers alternative code (lines 27–29).

----

## Concluding Remarks

There is a certain invariance among the various interprocess communication services. In other words, one service (or, more commonly, a combination of them) can accomplish the same task as another. For example, instead of using a channel, a static array could be created and data exchanged through it, employing mutual-exclusion semaphores to prevent concurrent access and event flags to notify a waiting process that data is ready. In some cases, such an implementation may prove more efficient, albeit less convenient.

Messages can be used for event synchronization instead of event flags&nbsp;– this approach makes sense when additional information needs to be transferred along with the flag. In fact, `OS::message` is specifically designed for this purpose. The variety of possible uses is vast, and the best choice for a given situation depends primarily on the specifics of that situation.

!!! info "**TIP**"

    It is important to understand and remember that any interprocess communication service performs its operations within a critical section (i.e., with interrupts disabled). Therefore, these services should not be overused where they can be avoided.

    For example, when accessing a static variable of a built-in type, using a mutual-exclusion semaphore is not a good idea compared to simply employing a critical section. A semaphore itself uses critical sections during locking and unlocking, and the time spent in them is longer than that required for direct variable access.

When using services in interrupt handlers, certain peculiarities arise. For instance, it is clearly a poor idea to call `TMutex::lock()` inside an interrupt handler: first, mutual-exclusion semaphores are intended for resource sharing at the process level, not the interrupt level; second, waiting for a resource to be released inside an ISR is impossible anyway and would only result in the interrupted process being placed into a waiting state at an inappropriate and unpredictable point. Effectively, the process would enter an inactive state from which it could only be extracted using `TBaseProcess::force_wake_up()`. In any case, nothing good would come of it.

A somewhat similar situation can occur when using channel objects in an interrupt handler. Waiting for data from a channel inside an ISR is impossible, with consequences analogous to those described above. Writing data to a channel is also not entirely safe: if insufficient space is available during a write, program behavior will deviate significantly from user expectations.

!!! warning "**RECOMMENDATION**"
    For operations inside interrupt handlers, use service member functions with the `_isr` suffix&nbsp;– these are specially designed versions that ensure both efficiency and safety when employing interprocess communication services within interrupts.

And, of course, if the existing set of interprocess communication services does not meet the needs of a particular project for any reason, it is always possible to design a custom service class tailored to specific requirements, building upon the provided base class `TService`. The standard set of services can serve as practical examples for such design.
