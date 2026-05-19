<!-- (C) 2026 Intel Corporation, MIT license -->
# Pattern triggers: source code

Use this file when you are reading source code and do not (yet) have profiling
data. Find the matching pattern below, then read the linked file in `patterns/`
for the full diagnosis and fix.

These patterns are worth fixing even without profiling confirmation — the code
structure alone is a strong predictor of the performance problem.

---

## Quick-match table

| Signal in source code | Pattern | Detail file |
|-----------------------|---------|-------------|
| Single FP variable updated in a loop: `sum += a[i]*b[i]`, `acc = fma(...)`, running max/min | Serial accumulator | `patterns/parallel-accumulator.md` |
| Inline asm or function uses `ymm`/`zmm0–15` registers with no `vzeroupper` before return or SSE call | Missing vzeroupper | `patterns/missing-vzeroupper.md` |
| `_mm_*` intrinsics (SSE/128-bit) or plain scalar float loop, no `_mm256_*` / `_mm512_*` | Narrow SIMD | `patterns/simd-upconversion.md` |
| C function with two or more pointer parameters, at least one written, no `restrict` qualifier, separate input/output buffers | Missing restrict | `patterns/missing-restrict.md` |
| Spinlock body is `while (!cmpxchg(&lock, ...))` with no prior read of the lock variable | Test-and-Set spinlock | `patterns/ttas.md` |
| Struct fields written by different threads, no `alignas(64)` between them | False sharing | `patterns/false-sharing.md` |
| Global `count++` / `atomic_inc` / `atomic_fetch_add` on a statistics field in a hot path | Shared statistics counter | `patterns/per-cpu-stats.md` |

---

## Pattern descriptions

### Missing vzeroupper

An inline assembly block or function that uses `ymm` registers or `zmm0`–`zmm15`
(which alias `ymm0`–`ymm15`) and returns or calls into code that may use legacy
SSE instructions, without emitting `vzeroupper` first. The upper 128 bits of the
YMM registers remain "dirty" from the CPU's perspective; the first SSE instruction
that follows will trigger an AVX↔SSE transition penalty costing hundreds of cycles.

Recognizable by: `ymm` or `zmm0`–`zmm15` in an asm block with no `vzeroupper`
at the end; or AVX intrinsic code (`_mm256_*`, `_mm512_*`) with no
`_mm256_zeroupper()` before returning to a caller compiled without AVX.

Read `patterns/missing-vzeroupper.md`.

---

### Serial accumulator

A loop that reduces a sequence into a single value — dot product, sum, running
max/min, weighted sum, histogram bucket — using one accumulator variable. The
accumulator is updated on every iteration (`sum += a[i] * b[i]`), creating a
loop-carried dependency: each iteration must wait for the previous one to finish
before it can begin. The CPU cannot exploit its ability to run multiple FP
operations per cycle. Recognizable by a single scalar variable on the left-hand
side of a compound-assignment inside the loop body, with no other loop-carried
dependency present.

Read `patterns/parallel-accumulator.md`.

---

### Missing restrict

A C function (not C++) that takes two or more pointer parameters — at least one
written — without `restrict`, where the buffers are guaranteed by the caller not
to overlap. Common signatures: `void filter(float *in, float *out, int n)`,
`void add(const float *a, const float *b, float *dst, int n)`. Without
`restrict`, the compiler must assume any write might alias any read of the same
type, forcing it to emit a runtime overlap check and a scalar fallback, or to
abandon auto-vectorization entirely.

Only applies to C. For C++, `__restrict__` (GCC/Clang extension) is available
but non-standard.

Read `patterns/missing-restrict.md`.

---

### Narrow SIMD

A floating-point or integer loop that uses 128-bit SSE intrinsics (`_mm_*`,
`__m128`, `__m128d`) or no SIMD at all (plain `float` or `double` arithmetic in
a loop). Modern x86 CPUs support 256-bit (AVX2) and often 512-bit (AVX-512)
operations that process 2–4× more data per instruction. This pattern also applies
when the compiler has auto-vectorized the loop but chose `xmm` registers —
visible in the object file or assembly listing (`-S` output). Check
`/proc/cpuinfo` for `avx2` and `avx512f` flags before recommending a target width.

Read `patterns/simd-upconversion.md`.

---

### Test-and-Set spinlock

A spinlock whose spin loop performs only an atomic read-modify-write operation
with no preceding ordinary read of the lock variable:

```c
while (!cmpxchg(&lock, UNLOCKED, LOCKED))   /* ◄ no read before the atomic */
    _mm_pause();
```

Every iteration of this loop acquires the lock's cache line exclusively — even
on failure — causing the line to bounce between all waiting threads and away from
the thread that holds the lock. Throughput degrades super-linearly with thread
count. The fix (TTAS) adds a cheap shared read before each atomic attempt.

Read `patterns/ttas.md`.

---

### False sharing

A struct contains fields that are written frequently by different threads (e.g.,
per-thread counters, state flags, or work-item metadata), and those fields are
not separated by `alignas(64)` / `__attribute__((aligned(64)))` padding. If they
land on the same 64-byte cache line, each write by one thread invalidates the
line for all other threads, even though no thread needs what the others wrote.
Recognizable by a shared struct definition where different fields are documented
as "owned by thread X" or updated inside per-thread loops with no explicit cache
line alignment between them.

Read `patterns/false-sharing.md`.

---

### Shared statistics counter

A global or shared counter that is incremented atomically from many threads in a
frequently-called path: `global_counter++`, `atomic_inc(&hits)`,
`atomic_fetch_add(&bytes, n)`. Unlike a lock, there is no retry logic — just a
bare increment — but the hardware still requires exclusive cache-line ownership
for every atomic write, causing the counter's cache line to bounce between all
updating threads. Field names are the strongest hint: `count`, `total`, `hits`,
`misses`, `errors`, `stat`, `bytes`, `packets`. No `cmpxchg` loop present
(if there is one, the TTAS pattern applies instead).

Read `patterns/per-cpu-stats.md`.
