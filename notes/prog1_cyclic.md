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
