# Job Queue

## Introduction

The job queue examined in this example is a message queue based on pointers to job objects. Traditionally, in OSes written in the C programming language, message queues are implemented using `void *` pointers combined with manual type casting. This approach is dictated by the facilities available in C. As previously discussed, this method is considered unsatisfactory due to concerns of convenience and safety. Therefore, a different approach will be used here—one made possible by the C++ language and offering several advantages.

First, there is no need for untyped pointers: the template mechanism allows efficient and safe use of pointers to concrete types, eliminating the need for manual type casting.

Second, flexibility of pointer-based messages can be further enhanced by allowing not only data transfer but also, in a sense, "exporting" actions—the message not only carries data but also enables specific actions to be performed at the receiving end of the queue. This is easily achieved through a hierarchy of polymorphic job classes[^1]. The approach described will be implemented in this example.

[^1]: For those new to C++ but familiar with C, an analogy can be drawn regarding technical implementation. The essence of polymorphism is performing different actions under the same description. C++ supports two kinds of polymorphism—static and dynamic. Static polymorphism is implemented via templates. Dynamic polymorphism is based on virtual functions. A hierarchy of polymorphic classes is built using dynamic polymorphism.

    Technically, the virtual function mechanism is implemented using tables of function pointers. Therefore, a similar mechanism could be implemented in C—for example, using structures containing pointers to arrays of function pointers. However, in C this would require much manual work, making it error-prone, less readable, labor-intensive, and inconvenient. C++ simply shifts all the routine work to the compiler, relieving the user from writing low-level code involving function pointer tables, their correct initialization, and usage.

Since only pointers are placed in the queue, the actual message payloads are located somewhere in memory. The placement method can vary—from static to dynamic; this aspect is omitted in the example, as it is not relevant to the discussion. In practice, the user decides based on task requirements, available resources, personal preferences, etc.

This example demonstrates a method of delegating job execution implemented using a message queue.

## Problem Statement

Developing virtually any program involves performing various actions, and these actions generally differ in importance and execution priority—which motivates the use of operating systems with priority-based schedulers. It often happens that, while handling events in a process, a need arises to perform an action requiring significant CPU time[^2] but without urgency—meaning it can quite reasonably be executed in a lower-priority process. In such cases, it is sensible not to delay the current process by performing the action directly, but to delegate its execution to another, lower-priority process.

[^2]: For example, extensive computations or updating the screen context in a program with a graphical user interface.

Moreover, such situations may occur multiple times in a program, and a logical solution is to create a dedicated low-priority process to which such jobs can be delegated from other processes when execution in high-priority processes is undesirable or impossible due to task constraints. The job transfer mechanism is conveniently implemented using polymorphic job classes and the `OS::channel` service as the transport for job objects.

## Implementation

All jobs—regardless of which process generated them or what specifically needs to be done—share one common property: they must be executed. This allows a mechanism where job execution can be launched in a unified way, while the actual job implementation is handled via virtual functions. To achieve this, an abstract base class is defined that specifies the job object interface:

```cpp
01    class Job
02    {
03    public:
04        virtual void execute() = 0;
05    };
```

Thus, there is a job object with its primary common property defined: it can be executed.

For brevity, two different types of time-consuming jobs[^3] will be considered:

[^3]: Obviously, this number can easily be increased if needed.

* computational—for example, evaluating a polynomial;
* transferring a significant amount of data—updating the screen buffer.

This requires defining two classes:

```cpp
01    class PolyVal : public Job
02    {
03    public:
04        virtual void execute();
05    };
06   
07    class UpdateScreen : public Job
08    {
09    public:
10        virtual void execute();
11    };
```

Objects of these classes will represent the jobs whose execution is delegated to the low-priority process. For details, see "Listing 1. Types and Objects in the Job Delegation Example".

```cpp
01    //---------------------------------------------------------------------
02    class Job // abstract job class
03    {
04    public:
05        virtual void execute() = 0;
06    };
07    //---------------------------------------------------------------------
08    class Polyval : public Job
09    {
10    public:
11        ... // constructors and the rest of the interface
12        virtual void execute();
13   
14    private:
15        ... // representation: polynomial coefficients,
16        ... // arguments,
17        ... // result, etc.
18    };
19   
20    //---------------------------------------------------------------------
21    class UpdateScreen : public Job
22    {
23    public:
24        ... // constructors and the rest of the interface
25        virtual void execute();
26   
27    private:
28        ... // representation
29    };
30    //---------------------------------------------------------------------
31    typedef OS::process<OS::pr1, 200> HighPriorityProc1;
32    ...
33    typedef OS::process<OS::pr3, 200> HighPriorityProc2;
34    ...
35    typedef OS::process<OS::pr7, 200> BackgroundProc;
36   
37    OS::channel<Job*, 4> job_queue;      // job queue with capacity for 4 elements
38    Polyval              poly_val;       // job object
39    UpdateScreen         update_screen;  // job object
40    ...
41    HighPriorityProc1    high_priority_proc1;
42    HighPriorityProc2    high_priority_proc2;
43    ...
44    BackgroundProc       background_proc;
45    //----------------------------------------------––-----------------------
```
/// Caption  
Listing 1. Types and Objects in the Job Delegation Example  
///

The abstract base class `Job` defines the job object interface; objects of this class cannot exist in the program. In this case, the interface is limited to a single function `execute()`, which enables the job to be run[^4]. Two concrete job classes—`Polyval` and `UpdateScreen`—are then defined, each targeted at specific goals: the first computes a polynomial value, the second updates the screen buffer.

[^4]: The interface can be extended with additional pure virtual functions if needed.

The remaining code is entirely standard—it follows the conventional C++ approach to defining types and objects, recommended for use with **scmRTOS**. Note that type definitions and object declarations can be placed in different files (headers and source files) as convenient for the project. Naturally, to avoid compilation errors, type definitions must be visible at points of object declaration—this is a standard requirement of C/C++.

The actual job delegation code based on the queue is shown below.

```cpp
01    //---------------------------------------------------------------------
02    template<> void HighPriorityProc1::exec()
03    {
04        const timeout_t DATA_UPDATE_PERIOD = 10;
05        for(;;)
06        {
07            ...
08            sleep(DATA_UPDATE_PERIOD);
09            ...                          // loading data into the job object
10            job_queue.push(&poly_val);   // placing the job into the queue
11        }
12    }
13    //---------------------------------------------------------------------
14    template<> void HighPriorityProc2::exec()
15    {
16        for(;;)
17        {
18            ...
19   
20            if(...) // screen element has changed
21            {
22                job_queue.push(&update_screen); // placing the job into the queue
23            }
24        }
25    }
26    //---------------------------------------------------------------------
27    template<> void BackgroundProc::exec()
28    {
29        for(;;)
30        {
31            Job *job;
32            job_queue.pop(job); // extracting a job from the queue
33            job->execute();     // executing the job
34        }
35    }
36    //---------------------------------------------------------------------
```
/// Caption  
Listing 2. Process Executable Functions  
///

In this example, two high-priority processes delegate part of their responsibilities to a lower-priority (background) process by placing jobs (with or without data[^5]) into a queue that the background process handles.

The background process itself "knows" nothing about what needs to be done for each job—its only responsibility is to launch the specified job, which contains sufficient information about what and how to do. The key point is that the delegated job executes with the appropriate (low, in this case) priority, without delaying high-priority processes[^6].

[^5]: A job may include any data that the sender places inside the job object.
[^6]: This applies not only to the processes delegating the job but also to other processes that might be blocked by lengthy job execution in high-priority processes.

Obviously, periodic background actions can easily be organized in the job-handling process. This is achieved by calling `pop()` with a timeout: upon timeout expiration, the process receives control, and the required actions can be performed at that moment. Coordinating these actions with job execution depends on project requirements and user decisions.

Technical aspects to note:

* Although the queue element type is a pointer to the base class `Job`, addresses of objects derived from `Job` are placed in the queue. This is crucial—it forms the basis for virtual function operation, central to polymorphic behavior. When `job->execute()` is called, the function belonging to the class of the object whose address was placed in the queue will actually be invoked;
* The job objects in the example are created statically. This is for simplicity—the creation method is irrelevant here; objects can be static or dynamically allocated, as long as they have non-local lifetime (i.e., persist between function calls). The existence of an active job is indicated not by the physical presence of the job object but by the presence of its address pointer in the queue.

Overall, the mechanism described is quite simple, has low overhead, and allows flexible distribution of program load across execution priorities.

!!! info "**NOTE**"
    The mechanism demonstrated above can be applied not only for executing low-priority jobs but also, conversely, for high-priority execution—relevant when a job requires urgency not provided by the originating process's priority. Technically, job transfer organization is identical to that described, with the only difference being that the job-handling process is a foreground[^7] rather than background process.

[^7]: Relative to the processes placing jobs in the queue.

#### Mutual-Exclusion Semaphores (Mutexes) and the Problem of Blocking High-Priority Processes

When discussing features of shared resource access from different processes via mutual-exclusion semaphores, a situation was described that is addressed by the [priority inheritance method](ipcs.md#mutex-priority-inversion).

The essence was that, under certain circumstances, a low-priority process can indirectly block a high-priority process. To solve this problem, the technique known as "priority inheritance" is often used: when a high-priority process attempts to acquire a mutex already held by a low-priority process, instead of simply waiting normally, the priorities are temporarily swapped until the mutex is released.

As previously noted, this method is not used in **scmRTOS** due to overhead comparable to (or greater than) the `TMutex` implementation itself.

To address the problem described, the technique presented in this example can be applied—only using a high-priority process as the job handler instead of a low-priority one. The program should be structured so that processes accessing shared resources do not perform the work themselves but delegate it as jobs to the high-priority handler process.

In this situation, no priority-related collisions arise, and the overhead of transferring jobs via pointers is negligible.
