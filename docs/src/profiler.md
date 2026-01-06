# Developing an Extension:<br>Process Profiler

## Purpose

The **process profiler** is an object responsible for collecting information about the relative active execution time of system processes, processing this data, and providing an interface through which user code can access the profiling results.

Collecting data on the relative execution time of processes can be accomplished in different ways—particularly through **sampling** the currently active process or by **measuring** the execution time of processes. In essence, the profiler class itself can remain the same; the choice between the two methods is implemented by organizing how the profiler interacts with OS objects and utilizes the processor's hardware resources.

Implementing the profiler class requires access to internal OS structures, but all such needs can be satisfied using the standard facilities provided by the operating system for exactly this purpose. Thus, a process activity profiler can be implemented as an **OS extension**.

The goal of this example is to demonstrate how a useful tool can be created that extends the operating system's functionality **without modifying the original OS source code**. Additional requirements:

* The developed class must not impose restrictions on how the profiler is used—the sampling period and points of use must be entirely determined by the user;
* The implementation should be as resource-efficient as possible, both in terms of executable code size and performance; in particular, the use of floating-point arithmetic must be avoided.

## Implementation

The profiler itself performs two main functions: collecting data on the relative active time of processes and processing this data to produce results.

Estimating process execution time can be based on a counter that accumulates this information. Accordingly, an array of such counters is needed for all system processes. An additional array is required to store the profiling results.

Thus, the profiler must contain two arrays of variables, a function to update the counters according to process activity, a function to process the counter values and store the results, and a function to access the profiling results. For greater flexibility, the profiler core is implemented as a template—see "Listing 1. Profiler".

```cpp
01 template <typename T>
02 class process_profiler : public OS::TKernelAgent
03 {
04     uint32_t time_interval();
05 public:
06     INLINE process_profiler();
07
08     INLINE void advance_counters()
09     {
10         uint32_t elapsed = time_interval();
11         counters[ cur_proc_priority() ] += elapsed;
12     }
13
14     INLINE T get_result(uint_fast8_t index) { return result[index]; }
15     INLINE void process_data();
16
17 protected:
18     volatile uint32_t counters[OS::PROCESS_COUNT];
19     T result [OS::PROCESS_COUNT];
20 };
```

/// Caption  
Listing 1. Profiler  
///

The profiler is implemented as a template, with the template parameter specifying the type of variables used for counters and results. This allows the most suitable type to be chosen for a specific application. It is assumed that the template parameter will be a numeric type—for example, `uint32_t` or `float`.

If the target platform has hardware support for floating-point operations, choosing `float` is preferable—such an implementation will likely be both faster and more compact. In the absence of such support, an integer type variant is more appropriate.

In addition to the elements listed above, there is a very important function `time_interval()` (line 4). The `time_interval()` function is defined by the user based on available resources and the chosen method of collecting process execution time data.

The call to `advance_counters()` must be organized by the user, and the placement of the call depends on the selected profiling method—statistical (sampling) or measurement-based.

The algorithm for processing collected data reduces to normalizing the counter values accumulated over the measurement period—see "Listing 2. Processing Profiling Results".

```cpp
01 template <typename T>
02 void process_profiler<T>::process_data()
03 {
04     // Use cache to make critical section as fast as possible
05     uint32_t counters_cache[OS::PROCESS_COUNT];
06
07     {
08         CritSect cs;
09         for(uint_fast8_t i = 0; i < OS::PROCESS_COUNT; ++i)
10         {
11             counters_cache[i] = counters[i];
12             counters[i] = 0;
13         }
14     }
15
16     uint32_t sum = 0;
17     for(uint_fast8_t i = 0; i < OS::PROCESS_COUNT; ++i)
18     {
19         sum += counters_cache[i];
20     }
21
22     for(uint_fast8_t i = 0; i < OS::PROCESS_COUNT; ++i)
23     {
24         if constexpr(std::is_integral_v<T>)
25         {
26             result[i] = static_cast<uint64_t>(counters_cache[i])*10000/sum;
27         }
28         else
29         {
30             result[i] = static_cast<T>(counters_cache[i])/sum*100;
31         }
32     }
33 }
```

/// Caption  
Listing 2. Processing Profiling Results  
///

The presented code, in particular, demonstrates how the result processing is selected depending on the template parameter type (line 24).

To avoid blocking interrupts for a significant time when accessing the counters array[^1], the array is copied into a temporary buffer, which is then used for further data processing.

[^1]: This access must be atomic to prevent corruption of the algorithm due to asynchronous modification of counter values by calls to `advance_counters()`.

When an integer type is chosen as the template parameter, the profiling result resolution is one hundredth of a percent, and the final results are stored in hundredths of a percent. This is achieved by normalizing each counter value—pre-multiplied by a coefficient defining the result resolution[^2]—to the sum of all counter values.

[^2]: In this case, the coefficient is 10000, which sets the resolution to 1/10000, corresponding to 0.01%.

This naturally imposes a limit on the maximum counter value used in calculations. For example, if the profiler counter variables are 32-bit unsigned integers (range \(0..2^{32}-1 = 0..4294967295\)) and multiplication by the coefficient 10000 is performed, the counter value must not exceed:

$$
N_{max} = \frac{2^{32} - 1}{10000} = \frac{4294967295}{10000} = 429496 \tag{1}
$$

This value is relatively small; to relax this constraint, calculations are performed with 64-bit precision—the counter value is cast to a 64-bit unsigned integer (line 26).

The user must also ensure that counters do not overflow during the profiling period, i.e., no accumulated counter value exceeds \(2^{32}-1\). This requirement is met by coordinating the profiling period with the magnitude of the value returned by `time_interval()`.

Integrating the profiler into a project is done by including the header file `profiler.h` in the project's configuration file `scmRTOS_extensions.h`.

### Usage

#### Statistical (Sampling) Method

For the statistical method, the call to `advance_counters()` should be placed in code that executes periodically at equal time intervals—for example, in an interrupt handler of a hardware timer. In **scmRTOS**, the system timer interrupt handler is well-suited for this purpose; the call to `advance_counters()` is placed in the user hook for the system timer, which must be enabled during configuration. In this case, `time_interval()` should always return 1.

#### Measurement Method

When choosing the measurement-based profiling method, `advance_counters()` must be called during context switches, which can be achieved by placing its call in the user hook for the context-switch interrupt.

Implementing `time_interval()` in this case is slightly more complex—the function must return a value proportional to the time interval between the previous and current calls. Measuring this interval requires utilizing some hardware resource of the target processor; in most cases, any hardware timer[^3] that allows reading its count register[^4] is suitable.

[^3]: Some processors (e.g., Blackfin or Cortex-A) include a dedicated CPU cycle counter that increments on every clock cycle, making time interval measurement very straightforward.
[^4]: For example, the WatchDog Timer in **MSP430** MCUs, while suitable as a system timer, is not appropriate for time interval measurement because the program cannot access its count register.

The scale of the value returned by `time_interval()` must be coordinated with the profiling period so that the sum of all values returned by this function for any process during the profiling period does not exceed \(2^{32}-1\)—see "Listing 3. Time Interval Measurement Function".

```cpp
01 template<typename T>
02 uint32_t process_profiler<T>::time_interval()
03 {
04     static uint32_t cycles;
05
06     uint32_t cyc = rpa(GTMR_CNT0_REG);
07     uint32_t res = cyc - cycles;
08     cycles = cyc;
09
10     return res;
11 }
```

/// Caption  
Listing 3. Time Interval Measurement Function  
///

In this example, a hardware CPU cycle counter running at 400 MHz is used (clock period 2.5 ns). The profiling period is chosen as 1 second. The ratio of periods results in the counter reaching:

$$
N = \frac{1}{2.5 \cdot 10^{-9}} = 400\,000\,000 \tag{2}
$$

This value is significantly less than \(2^{32}-1\), so no additional adjustments are needed. Otherwise, the function code would need modification to satisfy the condition.

Organizing the data collection period for relative active process time and displaying profiling results are left to the user.

For convenience, a user-defined class can be created to simplify usage by adding a result display function—see "Listing 4. User-Defined Profiler Class".

```cpp
01 class ProcProfiler : public process_profiler<float>
02 {
03 public:
04     ProcProfiler() {}
05     void get_results();
06 };
07 void ProcProfiler::get_results()
08 {
09     print("\n------------------------------\n");
10     print(" Pr | CPU, %% | Slack | Name\n");
11     print("------------------------------\n");
12
13 #if scmRTOS_DEBUG_ENABLE == 1
14     for(uint_fast8_t i = OS::PROCESS_COUNT; i ; )
15     {
16         --i;
17         float proc_busy;
18         if constexpr(std::is_integral_v<proc_profiler_data_t>)
19             proc_busy = proc_profiler.get_result(i)/100.0;
20         else
21             proc_busy = proc_profiler.get_result(i);
22
23         print(" %d | %7.4f | %4d | %s\n", i, proc_busy,
24               OS::get_proc(i)->stack_slack()*sizeof(stack_item_t),
25               OS::get_proc(i)->name() );
26     }
27 #endif
28
29     print("------------------------------\n\n");
30 }
```

/// Caption  
Listing 4. User-Defined Profiler Class  
///

Finally, it remains only to create an object of the class and ensure periodic calls to `process_data()`:

```cpp
ProcProfiler proc_profiler;
...
    ...
    proc_profiler.process_data(); // periodic call approx every 1 second
    ...
```
