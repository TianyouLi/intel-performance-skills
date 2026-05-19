<!-- (C) 2026 Intel Corporation, MIT license -->
# Pattern triggers: profiling output

Use this file when you have profiling data — `perf annotate`, `perf c2c`,
`perf stat`, `perf report`, VTune, flamegraphs, or any sampled CPU profile.
Find the matching pattern below, then read the linked file in `patterns/` for
the full diagnosis and fix.

---

## Quick-match table

| Signal in profile | Pattern | Detail file |
|-------------------|---------|-------------|
| Accumulate instruction (`vaddss`, `vaddpd`, `vfmadd*`) dominates annotate; IPC ≪ 1; cache misses low | Serial accumulator | `patterns/parallel-accumulator.md` |
| `other_assists.avx_to_sse` > 0 in `perf stat`; or first SSE instruction after AVX function carries extreme cycle count in annotate | Missing vzeroupper | `patterns/missing-vzeroupper.md` |
| Scalar preamble (pointer subtraction + conditional branch) before an otherwise vectorizable loop; or two loop versions (one vectorized, one scalar) in annotate for a simple array function | Missing restrict | `patterns/missing-restrict.md` |
| Scalar FP (`addsd`, `mulss`, `movsd`) or `xmm` registers in hot loop on AVX2/AVX-512 CPU | Narrow SIMD | `patterns/simd-upconversion.md` |
| `lock cmpxchg` cluster with high density in annotate; throughput drops with more threads | Test-and-Set spinlock | `patterns/ttas.md` |
| `perf c2c` HITM > 5% on a line; different byte offsets written by different threads | False sharing | `patterns/false-sharing.md` |
| `lock add` / `lock xadd` / `lock inc` in hot path; `perf c2c` true sharing on a stats/counter field | Shared statistics counter | `patterns/per-cpu-stats.md` |

---

## Pattern descriptions

### Missing restrict

`perf annotate` shows a scalar **aliasing preamble** before the main loop: a
pointer subtraction, comparison, and conditional branch to a scalar fallback,
even though the loop body is a simple array operation. Two versions of the same
loop may appear — one vectorized, one scalar — where the scalar path exists only
to handle the (never-occurring in practice) aliased case. The hot loop may also
be fully scalar if the compiler abandoned vectorization rather than emitting the
versioned form.

Read `patterns/missing-restrict.md`.

---

### Missing vzeroupper

Run `perf stat -e other_assists.avx_to_sse,other_assists.sse_to_avx ./program`.
A non-zero `other_assists.avx_to_sse` count confirms the penalty. In
`perf annotate`, the symptom is an extreme cycle count on the **first SSE
instruction** following a function that uses `ymm` or `zmm0`–`zmm15` registers —
the hardware is paying a transition penalty there. The fix is a single
`vzeroupper` instruction (or `_mm256_zeroupper()` intrinsic) before any exit
point that returns to SSE code.

Read `patterns/missing-vzeroupper.md`.

---

### Serial accumulator

The accumulate instruction (e.g., `vaddss`, `vaddpd`, `vmulss`, `vfmadd213ps`)
appears at the top of the `perf annotate` cycle-count column for a tight loop.
IPC from `perf stat` is well below 1.0, yet cache-miss rates are low — the CPU
is not waiting for memory, it is waiting for the previous iteration's result.
Cycles-per-iteration is at or above the FP latency of the operation (typically
4–5 cycles for `vadd`/`vfma`), even though the loop body is short.

Read `patterns/parallel-accumulator.md`.

---

### Narrow SIMD

`perf annotate` shows scalar floating-point instructions (`addsd`, `mulss`,
`movsd`, `vaddss`, `vmulss`) or 128-bit packed operations (`xmm` register names)
in the hot loop body, on a CPU that supports AVX2 (`ymm`) or AVX-512 (`zmm`).
The CPU can process 4–8× more data per instruction than it currently does. Also
applies when `perf stat` shows low IPC on a workload that is clearly CPU-bound
(low cache misses) — auto-vectorization may have produced narrow or scalar code.

Read `patterns/simd-upconversion.md`.

---

### Test-and-Set spinlock

`perf annotate` shows a tight cluster of `lock cmpxchg` (or `lock xchg`,
`lock cmpxchg8b`) instructions with high sample density, with no ordinary load
of the lock variable preceding the atomic. `perf report` shows `spin_lock`,
`cmpxchg`, `try_lock`, or CAS wrapper functions prominently. The defining
scaling signal: throughput **declines** (not just plateaus) as thread count
increases, or a dual-profile comparison shows these functions jumping from
invisible at 1 core to dominant at N cores.

Read `patterns/ttas.md`.

---

### False sharing

`perf c2c` reports HITM (Hit Modified) events above 5% on a cache line (`Tot
Hitm` column). The per-line access map shows **different byte offsets** being
written by different CPU or thread IDs — the key sign that threads are not
actually sharing data, just sharing a cache line. The hotspot function appears
in the multi-core profile but not (or barely) at 1 core, and the regression
scales linearly with thread count.

Read `patterns/false-sharing.md`.

---

### Shared statistics counter

`perf annotate` shows `lock add`, `lock xadd`, or `lock inc` in the hot path,
with no `cmpxchg` retry loop (which would indicate the TTAS pattern instead).
`perf c2c` shows true sharing on a counter or statistics field — the **same
byte offset** written by many CPUs. The function barely appears at 1 core but
grows linearly with thread count in the scaling profile, and field names suggest
accounting (`count`, `total`, `hits`, `misses`, `bytes`, `errors`, `stat`).

Read `patterns/per-cpu-stats.md`.
