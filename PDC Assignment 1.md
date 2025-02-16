## Problem 1: Threaded Mandelbrot Performance Analysis

### View 1

|**Threads**|**Serial Time (ms)**|**Threaded Time (ms)**|**Speedup**|
|---|---|---|---|
|2|652.889|337.431|1.93×|
|3|651.239|400.415|1.63×|
|4|652.862|277.969|2.35×|
|5|653.437|270.608|2.41×|
|6|653.320|215.159|3.04×|
|7|655.009|209.869|3.12×|
|8|652.150|215.328|3.03×|

**Observations (View 1):**

- **Non-linear Speedup:** Although increasing thread count generally reduces runtime, the speedup does not scale linearly. The best speedup is seen with 7 threads (3.12×).
- **Plateauing Effect:** Beyond 6 threads, additional threads yield diminishing returns (3.12× at 7 threads versus 3.03× at 8 threads).
- **Workload Distribution:** The dip in speedup for 3 threads indicates that dividing the work into three parts leads to an uneven load.

### View 2

|**Threads**|**Serial Time (ms)**|**Threaded Time (ms)**|**Speedup**|
|---|---|---|---|
|2|388.815|234.075|1.66×|
|3|388.144|179.465|2.16×|
|4|390.332|155.811|2.51×|
|5|390.612|138.645|2.82×|
|6|389.038|122.540|3.17×|
|7|390.416|111.100|3.51×|
|8|388.425|114.048|3.41×|

**Observations (View 2):**

- **Improved Speedup with More Threads:** The speedup increases with thread count, peaking at 7 threads (3.51×).
- **Diminishing Returns:** With 8 threads, the speedup slightly drops to 3.41×, likely due to overheads and contention.

**Hypothesis for Threaded Behavior:**

- **Workload Distribution & Imbalance:** Even when workload is divided, uneven portions (especially with 3 threads) cause bottlenecks.
- **Processor Architecture:** With four physical cores and hyper-threading (8 threads maximum), adding more threads forces sharing of core resources, which limits gains.
- **Synchronization Overhead:** Thread creation, synchronization, and context switching overhead increase as the thread count grows, leading to non-linear speedup.

---

## Problem 2: Vectorization Analysis

|**Vector Width**|**Total Vector Instructions**|**Vector Utilization (%)**|**Utilized Vector Lanes**|**Total Vector Lanes**|
|---|---|---|---|---|
|2|172,724|88.2|304,828|345,448|
|4|99,572|83.2|331,248|398,288|
|8|54,124|80.5|348,600|432,992|
|16|-|-|-|-|

_Observation:_  
As vector width increases, the total number of vector instructions decreases, and the utilization (i.e. the fraction of active lanes) declines slightly due to higher likelihood of masking (unused lanes) when the number of iterations isn’t a multiple of the vector width.

---

## Problem 3: Expected Speedup in Mandelbrot SIMD Execution

**Explanation:**

- The maximum expected speedup using 8-wide AVX2 SIMD instructions is **8×** in a perfect scenario.
- **Real-World Observation (~5× Speedup):**
    - **SIMD Divergence:** In the Mandelbrot computation, pixels require varying iteration counts (divergence). When some SIMD lanes exit early, the entire vector must wait for the slowest lane, reducing the overall speedup.
    - **Workload Imbalance:** Regions near fractal boundaries have non-uniform computation, lowering effective SIMD utilization.
    - **Overhead Costs:** Additional overhead (vector setup, masking) reduces the ideal 8× gain.

Thus, an observed ~5× speedup indicates that while the hardware supports 8-wide operations, inefficiencies due to divergence and imbalance prevent achieving the theoretical maximum.

_Reference:_  
[Optimizing Mandelbrot Generation with SIMD](https://bumbershootsoft.wordpress.com/2024/01/27/optimizing-mandelbrot-generation-with-simd/) citeturn0search9

---

## Problem 4: Iterative Square Root Speedup Analysis

### Data

```
[sqrt serial]:          [1176.217] ms
[sqrt ispc]:            [254.570] ms
[sqrt task ispc]:       [71.137] ms
```

### 4a. Speedup Breakdown

1. **SIMD Parallelization (Single-Core, No Tasks):**
    - **Serial:** 1176.217 ms
    - **ISPC (No Tasks):** 254.570 ms
    - **Speedup:** Serial time / ISPC time => 4.62×
2. **Multi-core Parallelization (All Cores, With Tasks):**
    - **ISPC with Tasks:** 71.137 ms
    - **Overall Speedup (vs. Serial):**  Serial time /  ISPC time => 16.53×
    - **Additional Multi-core Gain:** ISPC (no tasks) / ISPC (with tasks) => 3.58×

_Summary:_

- **SIMD speedup alone yields ~4.62× improvement.**
- **Multi-core tasking adds an additional ~3.58× gain, for an overall speedup of ~16.53× compared to the serial implementation.**

### 4b. Input Data Impact on Speedup

#### Maximum Speedup Case (Heavy, Uniform Workload)

```cpp
for (unsigned int i = 0; i < N; i++) {
    values[i] = 2.998f - i / (float)N;
}
```

**Why?**

- **Heavy Iteration Count:** Values close to 2.998f force the iterative method to run many iterations before convergence.
- **Uniformity:** The tiny variation (subtracting i/N) ensures that all SIMD lanes perform nearly the same amount of work, reducing divergence.
- **Amortized Overhead:** With many iterations, the fixed cost of SIMD setup becomes negligible relative to the total work.

_Observed Result:_

- **Serial Time:** ~2100 ms
- **ISPC (No Tasks) Time:** ~321 ms (~6.5× speedup)
- **ISPC with Tasks Time:** ~35 ms (~59.6× speedup overall)

#### Minimum Speedup Case (Light, Non-uniform Workload)

```cpp
for (unsigned int i = 0; i < N; i++) {
    values[i] = 1.0f;
}
```

**Why?**

- **Low Iteration Count:** When each value is 1.0f, the square root converges almost immediately (often in one iteration).
- **Fixed Overhead Dominates:** With minimal computation per element, the SIMD overhead (vector setup, loop control) becomes a large fraction of the total time.
- **Inefficient SIMD Utilization:** Any slight variation or masking due to divergence becomes more significant when the work is very light.

_Observed Result:_

- **Serial Time:** ~28 ms
- **ISPC (No Tasks) Time:** ~17 ms (~1.6× speedup)
- **ISPC with Tasks Time:** ~12 ms (~2.3× speedup overall)

---

