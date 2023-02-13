---
title: AFL instrumentation
description:
toc: true
authors: [xen134]
date: 2023-02-13T21:52:00+08:00
lastmod: 2023-02-13T21:52:00+08:00
draft: false
summary: Thoughts about instrumentation.
weight: 1
---

## Collection of coverage information in each run

The coverage is based on the number of traveled edges between basic blocks during execution. The instrumented assembly part `__afl_store` implements this mechanism.

The code snippet:

```assembly
  __afl_store:
    /* Calculate and store hit for the code location specified in rcx. */
    xorq __afl_prev_loc(%rip), %rcx
    xorq %rcx, __afl_prev_loc(%rip)
    shrq $1, __afl_prev_loc(%rip)
    incb (%rdx, %rcx, 1)
```

The equivalent pseudo C code is:

```c
void __afl_store(int64_t basic_block_id) {
    // global int64_t __afl_prev_loc
    edge_id = __afl_prev_loc ^ basic_block_id;
    __afl_prev_loc = (edge_id ^ __afl_prev_loc) >> 1;	// = basic_block_id >> 1
    *((uint8_t*)shm_ptr + edge_id)++;
}
```

*Here the double XOR is to save an register for preserving the original value of `%rcx`.

To distinguish the direction of the edge, the starting block id is shifted right for 1 bit while the ending block id is not touched. Then the corresponding byte of the edge in shared memory is incremented by 1 to denote that the path has been traveled once.

Later in function `has_new_bits()`, it will check the shared memory (aka. `trace_bits` in the C part) against the `virgin_bits` (local image of the shared memory) to see if any of the new paths have been added. After the checking `virgin_bits` will be synced with `trace_bits`.
