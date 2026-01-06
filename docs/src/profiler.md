# Extension Development:<br>Process Profiler

## Purpose

The process profiler is an object that performs the collection of information about the relative active time of system processes, processes this information, and has an interface through which a user program can access the profiling results.

Information about the relative process execution time can be collected in different ways&nbsp;– in particular, by sampling the current active process and by measuring process execution time. Essentially, the profiler class itself can be the same, with the choice of implementation for both methods made by organizing how the profiler interacts with OS objects and uses the processor's hardware resources.

The implementation of the profiler class itself requires access to the internals of the OS, but all these needs can be satisfied by the operating system's standard facilities, which are provided to the user for such purposes. Thus, the active process time profiler can be implemented as an OS extension.

The goal of this example is to show how to create a useful tool that extends the functional capabilities of the operating system without modifying the OS source code. Additional requirements:

  * The developed class must not impose restrictions on how the profiler is used&nbsp;– i.e., the information collection period and the place of use must be entirely determined by the user;
  * The implementation must be as resource-efficient as possible, both in terms of executable code size and performance, i.e., the use of floating-point calculations, in particular, must be excluded.

## Implementation

The profiler itself performs two main functions&nbsp;– collecting information about the relative active time of processes and processing this information to obtain results.

Estimating the process execution time can be implemented based on a counter that accumulates this information. Accordingly, for all system processes, an array of such counters is required. An array of variables storing the profiling results is also required.

Thus, the profiler must contain two arrays of variables: a function for updating the counters according to process activity, a function for processing the counter values and saving the results, and a function for accessing the profiling results. To increase flexibility of use, the base of the profiler is implemented as a template&nbsp;– see "Listing 1. Profiler".

```cpp
01    template <typename T>
02    class process_profiler : public OS::TKernelAgent
03    {
04        uint32_t time_interval();
05    public:
06        INLINE process_profiler();
07    
08        INLINE void advance_counters()
09        {
10            uint32_t elapsed = time_interval();
11            counters[ cur_proc_priority() ] += elapsed;
12        }
13    
14        INLINE T    get_result(uint_fast8_t index) { return result[index]; }
15        INLINE void process_data();
16    
17    protected:
18        volatile uint32_t  counters[OS::PROCESS_COUNT];
19                 T         result  [OS::PROCESS_COUNT];
20    };
```      
/// Caption
Listing 1. Profiler
///

The profiler is implemented as a template, whose parameter specifies the type of variables containing the counter and result values. This allows for selecting the most suitable option for a specific application. It is assumed that the template parameter will be a numeric type — for example, `uint32_t` or `float`.

If the target platform has hardware floating-point support, choosing `float` will be preferable — such an implementation will likely be both faster and more compact. In the absence of such support, using integer types would be advisable.

In addition to the above, there is a very important function `time_interval()` (4). The `time_interval()` function is defined by the user based on the available resources and the chosen method for collecting process execution time information.

The call to the `advance_counters()` function must be organized by the user, and the call site is determined by the chosen profiling method — statistical or measurement-based.

The algorithm for processing the collected information results in normalizing the counter values accumulated over the measurement period — see "Listing 2. Processing Profiling Results".

```cpp
01    template <typename T>
02    void process_profiler<T>::process_data()
03    {
04        // Use cache to make critical section fast as possible
05        uint32_t counters_cache[OS::PROCESS_COUNT];
06    
07        {
08            CritSect cs;
09            for(uint_fast8_t i = 0; i < OS::PROCESS_COUNT; ++i)
10            {
11                counters_cache[i] = counters[i];
12                counters[i]       = 0;
13            }
14        }
15    
16        uint32_t sum = 0;
17        for(uint_fast8_t i = 0; i < OS::PROCESS_COUNT; ++i)
18        {
19            sum += counters_cache[i];
20        }
21    
22        for(uint_fast8_t i = 0; i < OS::PROCESS_COUNT; ++i)
23        {
24            if constexpr(std::is_integral_v<T>)
25            {
26                result[i] = static_cast<uint64_t>(counters_cache[i])*10000/sum;
27            }
28            else
29            {
30                result[i] = static_cast<T>(counters_cache[i])/sum*100;
31            }
32        }
33    }
```
/// Caption
Listing 2. Processing Profiling Results
///

The provided code, in particular, shows how the result processing is selected depending on the template parameter type (24).

To avoid blocking interrupts for a significant amount of time while accessing the counters array[^1], this array is copied into a temporary array, which is then used for further data processing.

[^1]: This access must be made atomic to prevent corruption of the algorithm due to the possibility of asynchronous changes to the counter values when the `advance_counters()` function is called.

When an integer type is chosen as the template parameter, the resulting profiling resolution is one hundredth of a percent, and the final results are stored in hundredths of a percent. This is implemented by normalizing the value of each counter, previously multiplied by a coefficient defining the result resolution[^2], relative to the sum of the values of all counters.

[^2]: In this case, this coefficient is 10000, which sets the resolution to 1/10000, corresponding to 0.01%.

From these circumstances follows a natural limitation on the maximum value of the counter used in the calculations. For example, if the type of variables serving as the profiler's counters is a 32-bit unsigned integer, which can represent numbers in the range \(0..2^{32}-1 = 0..4294967295\), and the calculations involve multiplication by a coefficient of 10000, then to prevent overflow during calculations, the counter value must not exceed:

$$
N_{max} = \frac{2^{32} - 1}{10000} = \frac{4294967295}{10000} = 429496 \tag{1}
$$

The resulting value is relatively small, so to extend this limitation, the calculations are performed with 64-bit precision — the counter value is cast to a 64-bit unsigned integer (26).

The user must also ensure that no counter overflow occurs during the profiling period, i.e., the accumulated value in any counter does not exceed \(2^{32}-1\). This requirement is met by coordinating the profiling period and the resolution of the value returned by the `time_interval()` function.

The profiler is integrated into the project by including the header file `profiler.h` in the project's configuration file `scmRTOS_extensions.h`.

### Usage

#### Statistical Method

In the case of the statistical method, the call to the `advance_counters()` function should be placed in code that gains control periodically at equal time intervals — for example, in an interrupt handler of a timer; in **scmRTOS**, the system timer interrupt handler is well-suited for this purpose. In this case, the call to `advance_counters()` is placed in the user-defined system timer hook, whose invocation must be enabled during configuration. The `time_interval()` function in this case must always return 1.

#### Measurement Method

When choosing the measurement-based profiling method, the call to the `advance_counters()` function must occur during context switches. This can be achieved by placing its call in the user-defined context switch interrupt hook.

The implementation of the `time_interval()` function in this case becomes somewhat more complex — the function must return a value proportional to the time interval between its previous and current calls. Measuring this time interval requires utilizing certain hardware resources of the target processor. In most cases, any hardware timer[^3] that allows reading its timer register[^4] is suitable for this.

[^3]: Some processors, for example, Blackfin or Cortex-A, have a dedicated hardware counter for processor clock cycles, which increments on every clock cycle, allowing for very simple organization of time interval measurement.

[^4]: For instance, the WatchDog Timer of the **MSP430** MCU, which is quite suitable for use as a system timer, is not suitable for measuring time intervals because it does not allow the program to access its counting register.

The scale of the value returned by the `time_interval()` function must be coordinated with the profiling period so that the sum of all values returned by this function for any process over the profiling period does not exceed \(2^{32}-1\) — see "Listing 3. Time Interval Measurement Function".

```cpp
01    template<typename T>                         
02    uint32_t process_profiler<T>::time_interval()
03    {                                            
04        static uint32_t cycles;                  
05                                                 
06        uint32_t cyc = rpa(GTMR_CNT0_REG);       
07        uint32_t res = cyc - cycles;             
08        cycles       = cyc;                      
09                                                 
10        return res;                              
11    }                                            
```
/// Caption
Listing 3. Time Interval Measurement Function
///

In this example, a hardware processor clock cycle counter is used to measure time intervals. The processor operates at a frequency of 400 MHz, corresponding to a clock period of 2.5 ns. The profiling period is chosen to be 1 s. The ratio of the periods is such that the counter can reach the following value during the profiling period:

$$
N = \frac{2}{5 \cdot 10^{-9}} = 400 000 000 \tag{2}
$$

This value is significantly less than \(2^{32}-1\), so no additional actions are necessary. Otherwise, it would be necessary to modify the function code to ensure the specified condition is met.

Organizing the period for collecting information about the relative active time of processes and the method for displaying the profiling results are the user's responsibility.

For convenience, a user-defined class can be created to simplify usage by adding a function to display the results — see "Listing 4. User Profiler Class".

```cpp
01    class ProcProfiler : public process_profiler<float> 
02    {                                                                  
03    public:                                                            
04        ProcProfiler() {}                                              
05        void get_results();                                            
06    };                                                                 

07    void ProcProfiler::get_results()                                      
08    {                                                                     
09        print("\n------------------------------\n");                      
10        print(" Pr |  CPU, %% | Slack | Name\n");                         
11        print("------------------------------\n");                        
12                                                                          
13        #if scmRTOS_DEBUG_ENABLE == 1                                     
14        for(uint_fast8_t i = OS::PROCESS_COUNT; i ; )                     
15        {                                                                 
16            --i;                                                          
17            float proc_busy;                                              
18            if constexpr(std::is_integral_v<proc_profiler_data_t>)        
19                proc_busy = proc_profiler.get_result(i)/100.0;            
20            else                                                          
21                proc_busy = proc_profiler.get_result(i);                  
22                                                                          
23            print(" %d  | %7.4f | %4d  | %s\n", i, proc_busy,             
24                      OS::get_proc(i)->stack_slack()*sizeof(stack_item_t),
25                      OS::get_proc(i)->name() );                          
26        }                                                                 
27        #endif                                                            
28                                                                          
29        print("------------------------------\n\n");                      
30    }                                                                     
```
/// Caption
Listing 4. User Profiler Class
///

Finally, all that remains is to create an object of the class and ensure the periodic calling of the `process_data()` function:

```cpp
ProcProfiler proc_profiler;
...

    ...
    proc_profiler.process_data();  // periodic call approx every 1 second
    ...

```
