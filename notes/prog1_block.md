# Program 1 - BLOCK decomposition (imbalanced baseline)
Machine: i7-8650U, 4 physical cores / 8 threads, VMware VM
All runs: view 1 unless stated, 5 iterations per run, harness reports min

| Threads | Speedup | Min per-thread ms | Max per-thread ms |
|---------|---------|-------------------|-------------------|
| 2       | 1.90x   | 220.989           | 223.014           |
| 3       | 1.54x   |  91.578           | 277.214           |
| 4       | 2.17x   |  51.217           | 223.934           |
| 8       | 3.38x   |  10.521           | 127.664           |

View 2, 4 threads: 2.30x speedup, min 55.281 ms, max 115.930 ms
  (imbalance inverted: thread 0 slowest, not the middle threads)

Serial baseline view 1: ~424-451 ms
Serial baseline view 2: ~252 ms

KEY FINDING: 3 threads (1.54x) is SLOWER than 2 threads (1.90x).
Thread 1 owns rows 400-799 = the entire expensive band.
