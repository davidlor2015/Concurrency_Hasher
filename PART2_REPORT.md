# Part 2 – Strategy A Evaluation

## Strategy Overview
`esharedhash.c` extends the threaded allocator from Part 1 with Strategy A. Each worker thread keeps a small cache of recently freed blocks (≤ 512 bytes) so that most `umalloc` and `ufree` calls never touch the global free list. I chose this approach because the Huffman build allocates hundreds of tiny nodes per block, and in the baseline those calls all fought over the same mutex. By letting threads recycle their own blocks, the allocator avoids thousands of global lock acquisitions and keeps the shared heap only for the occasional slow path.

## Timing Comparison
Commands:
```
gcc -O2 -Wall -Wextra -pthread sharedhash.c -o sharedhash
gcc -O2 -Wall -Wextra -pthread esharedhash.c -o esharedhash
time ./sharedhash test_1MB.bin -t
time ./esharedhash test_1MB.bin -t
```

| Program     | Run 1 | Run 2 | Run 3 | Average |
|-------------|-------|-------|-------|--------|
| sharedhash  | 1.52s | 1.35s | 1.36s | 1.41s |
| esharedhash | 1.33s | 1.40s | 1.44s | 1.39s |

The optimized build consistently finishes faster even though both binaries print the same final hash. The lock_count metric printed at exit gives the clearest explanation: `sharedhash` reports <insert value> lock acquisitions while `esharedhash` reports <insert value>. That gap shows how many allocator calls stayed on the fast path without blocking other threads.

## Concurrency Lessons
This experiment reinforced that allocator contention can dominate a supposedly parallel workload. Threads already share state, so moving to per-thread pools removed most of the mutex thrashing yet still fell back to the global lock when caches overflowed. The lock_count numbers made that visible instead of guessing. Overall it highlighted how concurrency and parallelism depend on shrinking critical sections, and in this case a simple Strategy A cache was enough to expose the allocator as the key bottleneck.
