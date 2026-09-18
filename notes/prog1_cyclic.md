# Program 1 - ROW-CYCLIC decomposition (balanced)
Thread i computes rows i, i+numThreads, i+2*numThreads, ...

| Threads | Speedup | Min ms  | Max ms  | Max/Min |
|---------|---------|---------|---------|---------|
| 2       | 1.96x   | 216.450 | 216.432 | 1.00    |
| 3       | 2.69x   | 157.822 | 161.426 | 1.02    |
| 4       | 3.24x   | 135.091 | 138.278 | 1.02    |
| 8       | 5.18x   |  70.177 |  85.868 | 1.21    |

View 2, 4 threads: 3.32x, min 69.470 ms, max 76.799 ms (ratio 1.11)

COMPARISON (block -> cyclic):
  2 threads: 1.90x -> 1.96x
  3 threads: 1.54x -> 2.69x   (anomaly removed)
  4 threads: 2.17x -> 3.24x   (imbalance ratio 4.37 -> 1.02)
  8 threads: 3.38x -> 5.18x
  view2 4t:  2.30x -> 3.32x

8-thread gap vs ideal 8x explained by:
  (a) only C=4 physical cores; threads 8 = 4 cores x 2 SMT siblings
      sharing FP units. 4t=3.24x -> 8t=5.18x is only 1.60x for 2x threads.
  (b) residual imbalance: spread widens to 1.21 at stride 8
  (c) VMware guest cannot see SMT topology; all-core turbo on a 15W
      U-series chip is well below single-core turbo used by the serial run

## Part 5: 2T threads (2 x 8 = 16)
16 threads: 5.15x  (vs 5.18x at 8 threads) - no improvement
  per-thread: 75 rows each, ~34-87 ms, spread ratio ~2.5
  8 threads:  150 rows each, ~70-86 ms, spread ratio 1.21

Conclusion: hardware already saturated at T=8. Adding software threads
adds no execution resources, only queueing. Each thread does half the
work at roughly half speed -> same wall clock.
Wider spread at 16t is SCHEDULING JITTER, not load imbalance: every
thread has exactly 75 rows of comparable cost. Threads descheduled
mid-row record longer wall times for identical work.
No slowdown because Mandelbrot has a tiny working set and long-running
rows, so context switches are rare and caches stay warm.
