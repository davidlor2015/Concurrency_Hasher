# Concurrent File Hasher (Threaded Allocator Project)

## Overview

The concurrent file hasher ingests an arbitrary binary input and slices it into fixed-size blocks that can be scheduled independently. Each block is handed off to a worker that can run in single-threaded, multi-process, or multi-threaded form depending on the execution mode selected at launch. The block hashes are reassembled into a deterministic final signature to capture the entire file state. All heap interactions are funneled through a custom userspace allocator (`umalloc`/`ufree`) that replaces the default `malloc`/`free`, enabling tight control over memory usage patterns and experimentation with allocator policies.

## Features

- Single-threaded mode for baseline correctness validation and deterministic debugging.
- Legacy multi-process mode (`-m`) that stages workers across processes for isolation.
- Multi-threaded mode (`-t`) that keeps scheduling overhead low by sharing address space.
- Custom allocator that provides `umalloc`/`ufree` so hashing logic never touches `malloc`.
- Thread-safe synchronization primitives to coordinate producers, workers, and reducers.
- Experimental Part 2 allocator optimizations that explore Strategy A heuristics for reduced contention.

## File Descriptions

- `sharedhash.c` – Part 1 implementation of the threaded hasher plus the baseline allocator wrapper
- `esharedhash.c` – Part 2 variant that layers Strategy A allocator optimizations on the shared hasher
- `report.md` – Part 1 design write-up
- `part2_report.md` – Part 2 allocator performance analysis

## Compilation

```sh
# Build the Part 1 threaded hasher
make sharedhash

# Build the Part 2 Strategy A variant
make esharedhash

# Clean intermediate objects
make clean
```
