# Shared Hash Assignment Report

## Part 1 - Threaded Baseline

### Implementation Summary
`sharedhash.c` now spawns one `pthread` per 1 KB block (bounded by `MAX_THREADS` = 16). The parent thread copies each block into shared memory via `umalloc`, launches a worker that calls `process_block`, and stores that block's hash into a shared `results[]` array. Once all workers complete (`pthread_join`), the parent folds the hashes in block order so the final signature matches the single-thread baseline exactly. All IPC (pipes) and process management (`fork`/`waitpid`) were removed.

Allocator synchronization no longer relies on a process-shared semaphore. Instead, the existing `pthread_mutex_t mLock` is initialized in `init_umem` when `-t` is passed, and the global allocator wrappers lock/unlock around `_umalloc`/`_ufree`. In the old multi-process design the allocator state had to live in `mmap`-backed shared memory so that independent processes could see it; in the threaded version every worker naturally shares the same address space, so the pointer-rich allocator structures can stay in-process and we only need an intra-process mutex. This eliminates the double-indirection hacks that the process version required for correctness.

### Part 1 — Brief Explanation of Changes
For Part 1 I rewrote `sharedhash.c` so it runs in a single process with pthreads instead of spawning separate worker processes. All of the `fork`, `pipe`, and shared-memory plumbing went away; each 1 KB block is now handed to a `pthread_create` worker that writes its hash directly into a shared `results[]` array. Since threads already share globals and heap state, `mmap` and process semaphores were replaced with normal `malloc` and a scoped `pthread_mutex_t` guarding the allocator.

Thread-based memory sharing is a lot simpler because every worker runs in the same address space. In the process version I had to set up shared memory and synchronize with semaphores and pipes, whereas the threaded conversion just needs a mutex to keep the allocator safe. That change cut down a ton of boilerplate and makes it easier to reason about the flow, as long as I am disciplined about locking the shared sections.

### Build Commands
```
gcc -DDEBUG=2 -Wall -Wextra -pthread sharedhash.c -o sharedhash
gcc -Wall -Wextra -pthread esharedhash.c -o esharedhash   # reference/optimized build
```

### Correctness Evidence
All three execution modes (sequential baseline, legacy multi-process build supplied by the course staff, and the new thread mode) produce the same final signature for every test file tried. Representative runs are below; the process log was captured from the archived multi-process executable and is kept only for comparison.

| Mode | Command | Final signature |
| --- | --- | --- |
| Single-thread (`run_single`) | `wsl sharedhash pattern.bin` | `1440507007` |
| Threaded (`run_multi`) | `wsl sharedhash pattern.bin -t` | `1440507007` |
| Process baseline (archival) | `sharedhash_mproc pattern.bin` | `1440507007` |

Additional spot checks on `tiny.bin` and `test_1MB.bin` yielded identical final hashes across modes:

```
$ wsl sharedhash tiny.bin
[PID 1254] Block 0 hash: 2832194
Final signature: 2832194

$ wsl sharedhash tiny.bin -t
[PID 1261] Block 0 hash: 2832194
Final signature: 2832194

$ wsl sharedhash test_1MB.bin
...
Final signature: 365586242

$ wsl sharedhash test_1MB.bin -t
...
Final signature: 365586242
```

Because the final reduction step is identical and the allocator remains memory safe (guarded by `pthread_mutex_t`), the thread mode preserves the final signature that the multi-process reference produced for every file.




