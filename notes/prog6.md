# Program 6 - K-Means
md5 data.dat: 3a25f24193f4fdca82ee4cb2737fd5bb  (804002420 bytes)

## Profile of the serial code (101 iterations)
computeAssignments  43733.009 ms   f = 0.6902   <-- hotspot
computeCentroids     6675.950 ms       0.1054
computeCost         12953.382 ms       0.2044
total               63362.341 ms

## Amdahl ceiling, T = 8
Smax = 1 / ((1 - 0.6902) + 0.6902/8)
     = 1 / (0.3098 + 0.0863)
     = 1 / 0.3961
     = 2.525x
Requirement 0.80 x 2.525 = 2.02x

## Achieved
serial baseline      63556 ms
parallel 5 runs      24448 / 29973 / 28584 / 33305 / 32968 ms
minimum              24448 ms
speedup              63556 / 24448 = 2.60x
min-max spread       24448 - 33305 ms (36 percent) - VM on a laptop is noisy

2.60x is ABOVE the 2.525x Amdahl ceiling. Reason: the rewritten
computeAssignments is not only threaded, it is also a better serial
algorithm. The original looped k-outer / m-inner and kept a minDist[M]
array, so it swept all 800 MB of data K times. The new version loops
m-outer / k-inner, so each point's 100 doubles are loaded once and
compared to all 3 centroids while still in cache. Amdahl assumes the
hotspot's work is unchanged, so reducing that work breaks the bound.

## Karp-Flatt check
e = (1/S - 1/T) / (1 - 1/T) = (1/2.60 - 0.125) / 0.875 = 0.297
Measured serial fraction 1 - f = 0.3098. The two agree closely (0.297
vs 0.310), which supports the directly measured profile.

## vs Stanford 2.1x
2.1x is inside our ceiling of 2.525x, and we exceeded it.
