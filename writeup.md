# CS3006 Assignment 2 — Performance Analysis on a Multi-Core CPU

**Name:** Muhammad Qudama
**Roll Number:** 23L-0852
**Date:** 20 September 2026

---

## 1. Machine Declaration

**This assignment was run inside a VMware Workstation virtual machine on a Windows host.**

| Field | Value |
|---|---|
| CPU model | Intel Core i7-8650U @ 1.90 GHz (Lenovo ThinkPad T480s) |
| Physical cores (C) | 4 |
| Hardware threads (T) | 8 |
| SMT present? | Yes on host silicon; NOT visible to guest |
| Base / max clock | 1.90 GHz / 4.20 GHz |
| Widest SIMD (W) | AVX2, 8-wide single precision |
| RAM | 8 GB allocated to VM; host 16 GB LPDDR3-2133 dual channel, soldered |
| Machine type | VMware Workstation VM, Windows host, own laptop |
| OS / kernel | Ubuntu 26.04 "resolute", kernel 7.0.0-31-generic |
| GCC | 15.2.0 |
| ISPC | 1.31.0 (LLVM 23.0.0) |
| Power state | Plugged in, Windows "Best performance" |

Derived constants: **C = 4, T = 8, W = 8**. Program 3 part 2 ceiling = C x W = **32**.

### Note on virtualised topology

The guest reports `Thread(s) per core: 1`, `Core(s) per socket: 2`, `Socket(s): 4`,
and `/sys/devices/system/cpu/cpu0/topology/thread_siblings_list` returns `0`.

[WRITE: Explain that VMware presents a fake topology, why you still use C=4
and not C=8, and what this means for the hyper-threading observations in
Program 1.]

### Measurement protocol

I followed the Section 5.2 protocol: [WRITE: state what you actually did —
closed other applications, plugged in, ran each configuration 5 times and
report the minimum, report min and max spread, let the machine cool between
long runs. Be honest about anything you could not do, e.g. cpupower governor
inside a VM.]

---

## 2. Build Notes

- Added `#include <cstring>` to `prog1_mandelbrot_threads/main.cpp`
- Added `#include <cstdlib>` to `prog1_mandelbrot_threads/mandelbrotThread.cpp`
- ISPC v1.31.0 as required by the addendum (not Stanford's v1.28.1)
- Programs 2-6 built cleanly as shipped
- No additional fixes were needed despite GCC 15.2 being newer than the
  GCC 13 the addendum targets
- `prog6_kmeans/requirements.txt` pin ignored; installed current
  matplotlib 3.11.2 and scikit-learn 1.9.1 per addendum Section 7.4

---

## 3. Program 1 — Mandelbrot with Threads

### 3.1 Block decomposition (first attempt)

| Threads | Speedup | Min per-thread (ms) | Max per-thread (ms) | Max/Min |
|---|---|---|---|---|
| 2 | 1.90x | 220.989 | 223.014 | 1.01 |
| 3 | 1.54x | 91.578 | 277.214 | 3.03 |
| 4 | 2.17x | 51.217 | 223.934 | 4.37 |
| 8 | 3.38x | 10.521 | 127.664 | 12.61 |

View 2, 4 threads: 2.30x, min 55.281 ms, max 115.930 ms (ratio 2.10)
Serial baseline: view 1 ~424-451 ms, view 2 ~252 ms

### 3.2 Why speedup is not linear

[WRITE: The key finding is that 3 threads (1.54x) is SLOWER than 2 threads
(1.90x). Explain using the per-thread times above: where the expensive
pixels are, why the 2-thread split happens to be balanced, why thread 1
at 3 threads gets the whole expensive band, and why wall-clock is set by
the slowest thread. Mention that view 2 shows the same flaw with the hot
region in a different place.]

### 3.3 Row-cyclic decomposition (the fix)

Thread i computes rows i, i+numThreads, i+2*numThreads, ...

| Threads | Speedup | Min (ms) | Max (ms) | Max/Min |
|---|---|---|---|---|
| 2 | 1.96x | 216.450 | 216.432 | 1.00 |
| 3 | 2.69x | 157.822 | 161.426 | 1.02 |
| 4 | 3.24x | 135.091 | 138.278 | 1.02 |
| 8 | 5.18x | 70.177 | 85.868 | 1.21 |

View 2, 4 threads: 3.32x, min 69.470 ms, max 76.799 ms (ratio 1.11)

**Before vs after:** 2t 1.90 -> 1.96, 3t 1.54 -> 2.69, 4t 2.17 -> 3.24,
8t 3.38 -> 5.18, view2 4t 2.30 -> 3.32

[WRITE: Explain why interleaving works. Point at the Max/Min column as the
evidence: 4.37 -> 1.02 at 4 threads.]

### 3.4 Speedup at T threads vs ideal T

Achieved **5.18x** at T = 8 against an ideal of 8x (65% efficiency).

[WRITE: Explain the gap with three SEPARATE causes:
 (a) only C=4 physical cores, SMT siblings share FP units — support this
     with your own numbers, 4t=3.24x to 8t=5.18x is only 1.60x for double
     the threads
 (b) residual imbalance, spread widens from 1.02 to 1.21 at stride 8
 (c) virtualisation and all-core turbo being lower than single-core turbo
     on a 15W U-series chip]

### 3.5 Part 5 — running with 2T = 16 threads

16 threads: **5.15x**, versus 5.18x at 8 threads. No improvement.
Per-thread spread at 16t is ~34-87 ms (ratio ~2.5) vs 1.21 at 8t.

[WRITE: Explain why more software threads than hardware threads gives
nothing. Note that each thread does half the work (75 rows vs 150) in
roughly the same time, which shows each ran at about half speed. Explain
that the wider spread is scheduling jitter, NOT load imbalance, since
every thread has exactly 75 rows of comparable cost. Say why there is no
slowdown either.]

---

## 4. Program 2 — Vector Intrinsics

Workload size N = 10000.

| VECTOR_WIDTH | Utilization | Total vector instructions |
|---|---|---|
| 2 | 77.9% | 167,727 |
| 4 | 70.7% | 97,075 |
| 8 | 67.0% | 52,877 |
| 16 | 65.2% | 27,592 |

CLAMPED EXPONENT and ARRAY SUM both passed at every width.

[WRITE: Explain the downward trend. The while(count>0) loop runs until the
largest exponent in the gang is exhausted; exponents are random in 0..9;
the wider the vector the more likely at least one lane holds a 9, so more
lanes sit masked off while still occupying the vector. Connect this to the
general principle about SIMD divergence.]

---

## 5. Program 3 — Mandelbrot with ISPC

### 5.1 Part 1: SIMD only. Ceiling = W = 8

View 1: **4.59x**   View 2: **4.12x**

[WRITE: State the ceiling is W=8, then explain why you observe less.
Mention lane divergence in mandel() — lanes that escape early are masked
but still occupy the gang until the slowest lane finishes. Also mention
that the Makefile uses --opt=disable-fma and -ffp-contract=off to keep
serial and ISPC bit-identical, which costs throughput.]

### 5.2 Part 2: with tasks. Ceiling = C x W = 4 x 8 = 32

Task count sweep, view 1, 3 runs each (best of 3 shown):

| Tasks | Best speedup from task ISPC |
|---|---|
| 2 | 10.22x |
| 4 | 13.12x |
| 8 | 17.77x |
| 16 | 24.70x |

Final 5 runs at NUM_TASKS = 16:
- ISPC only: 5.65 / 5.54 / 5.04 / 5.44 / 4.62 — best **5.65x**
- Task ISPC: 30.09 / 25.79 / 27.22 / 22.15 / 24.76 — best **30.09x**, spread 22.15-30.09

**Achieved 30.09x of the 32x ceiling = 94%.**

[WRITE: Report your achieved speedup as a fraction of C x W and account for
the shortfall. Then explain the more interesting question: why 16 tasks
beats 8 even though there are only 8 hardware threads. Connect it to the
load-imbalance problem you solved by hand in Program 1.]

**Methodology note:** [WRITE: Mention honestly that your first sweep was
invalid because you omitted the --tasks flag, so every task count gave the
same ~51 ms. Say how you noticed and what you did. This is exactly the kind
of measurement-that-changed-your-mind the addendum asks for.]

---

## 6. Program 4 — sqrt

| Input | Serial (ms) | ISPC (ms) | Task ISPC (ms) | ISPC speedup | Task speedup |
|---|---|---|---|---|---|
| Random (starter) | 1141.7 | 314.0 | 52.1 | 3.64x | 21.92x |
| Max-speedup input | 2754.3 | 422.1 | 94.4 | **6.53x** | 29.17x |
| Min-speedup input | 269.7 | 322.4 | 59.8 | **0.84x** | 4.51x |

**Max input:** every element set to 2.998f
**Min input:** `values[i] = (i % 8 == 0) ? 2.998f : 1.0f`

[WRITE: Reasoning for each input. For MAX: why making every lane take the
same long path maximises utilisation. For MIN: why one expensive element
per 8-wide gang is the worst case, and why ISPC comes out SLOWER than
serial at 0.84x. Also explain why the task version still wins in both
cases.]

---

## 7. Program 5 — saxpy

5 runs (ISPC): 24.910 / 14.557 / 14.892 / 16.786 / 14.702 ms
Best: **14.557 ms = 20.473 GB/s**. Spread 11.96 - 20.47 GB/s.
Tasks: 0.99x and 0.82x — no benefit.

**Theoretical peak bandwidth:** LPDDR3-2133, dual channel, soldered (ThinkPad T480s)
2 channels x 8 bytes x 2133 MT/s = **34.1 GB/s**

**Measured 20.5 GB/s = 60% of theoretical peak.**

[WRITE: Explain the performance. saxpy moves 16 bytes per 2 FLOPs, so it is
memory bound, not compute bound. Explain why tasks do not help. Comment on
whether 60% of theoretical peak is reasonable.]

---

## 8. Program 6 — K-Means

**md5sum of data.dat:** `3a25f24193f4fdca82ee4cb2737fd5bb`
**Size:** 804,002,420 bytes (766.8 MiB) — matches the addendum

### 8.1 Profile of the serial code (101 iterations)

| Function | Time (ms) | Fraction |
|---|---|---|
| computeAssignments | 43,733.009 | **0.6902** |
| computeCentroids | 6,675.950 | 0.1054 |
| computeCost | 12,953.382 | 0.2044 |
| Total | 63,362.341 | |

**f = 0.6902**, hotspot is `computeAssignments`.

### 8.2 Amdahl ceiling

    Smax = 1 / ((1 - f) + f/T)
         = 1 / ((1 - 0.6902) + 0.6902/8)
         = 1 / (0.3098 + 0.0863)
         = 1 / 0.3961
         = 2.525x

Requirement: 0.80 x 2.525 = **2.02x**

### 8.3 Achieved

Serial baseline: 63,556 ms
Parallel, 5 runs: 24,448 / 29,973 / 28,584 / 33,305 / 32,968 ms
Minimum: **24,448 ms**
Speedup: 63,556 / 24,448 = **2.60x**
Spread: 24,448 - 33,305 ms (36%)

### 8.4 Profiling narrative

[WRITE: Tell the story. What you measured, what it led you to believe,
what you changed, what happened. Include the two changes you made to
computeAssignments: the loop order swap (k-outer/m-inner to m-outer/
k-inner) and the threading across data points. Explain why the loop swap
matters — the original swept 800 MB of data K times and kept a minDist[M]
array; the new one loads each point's 100 doubles once.]

### 8.5 Exceeding the Amdahl ceiling

2.60x is **above** the computed 2.525x ceiling.

[WRITE: Explain why this is not an error. Amdahl's law assumes the work in
the hotspot is unchanged and only spread across threads. Your rewrite also
reduced the serial work through better cache behaviour, so the bound does
not apply. This is worth a careful paragraph.]

### 8.6 Karp-Flatt check

    e = (1/S - 1/T) / (1 - 1/T)
      = (1/2.60 - 0.125) / 0.875
      = (0.3846 - 0.125) / 0.875
      = 0.297

Measured serial fraction (1 - f) = 0.3098.

[WRITE: Compare 0.297 against 0.3098 and say whether they agree, and what
that agreement tells you about the reliability of your direct profile.]

### 8.7 Relative to the Stanford 2.1x target

[WRITE: State where you sit relative to 2.1x and whether it was inside or
above your machine's ceiling of 2.525x.]

### 8.8 Plots

`plots/start.png` and `plots/end.png` are included.

[WRITE: One or two sentences confirming end.png shows three coherent
clusters with centroids sitting in them. Note the addendum's warning that
100-dimensional points projected to 2D by PCA can look misassigned while
being correct in the real space.]

---

## 9. Extra Credit

Not attempted.

---

## 10. Measurement Honesty Statement

[WRITE: One short paragraph confirming every number above was measured by
you on the declared machine, and noting the main limitation — a VM on a
15W laptop is a noisy instrument, as the 36% spread in Program 6 and the
22-30x spread in Program 3 both show.]
