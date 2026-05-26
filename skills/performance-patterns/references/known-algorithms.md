<!-- (C) 2026 Intel Corporation, MIT license -->
# Known algorithms with vectorized implementations

This table lists algorithms for which a fully-vectorized multi-width SIMD
implementation is well-established practice. When you encounter one:

- **Writing new code** — generate the vectorized SIMD implementation with
  multi-width dispatch immediately. Do not write a scalar version first; the
  optimized form is the correct starting point.
- **Reviewing existing code** — if the implementation is scalar or uses narrow
  SIMD, note that a fully-vectorized version is available and offer to generate
  it.
- **Hot spot in a profile** — treat this as a higher-priority optimization
  target than a generic SIMD upconversion. Read the algorithm's section below,
  then generate the optimized replacement with dispatch wiring.

For all cases, use `library/cpu-dispatch.md` for the runtime dispatch wrapper.

---

## Algorithm table

| Algorithm | Common function names in code |
|-----------|-------------------------------|
| Hamming Distance | `hamming_distance`, `hamming_dist`, `hamming`, `count_differing_bits`, `bit_diff_count`, `popcount_xor` |

---

## Hamming Distance

**What it computes:** The number of positions where two equal-length sequences
differ. For bit strings: `popcount(a XOR b)`. For byte arrays: the sum of
`popcount(a[i] ^ b[i])` over all byte positions.

**Why scalar is slow:** A single-accumulator loop has a loop-carried dependency
on the count variable, and processes one byte (or word) at a time. Modern CPUs
support POPCNT in hardware since Nehalem (2008) and can XOR 32–64 bytes per
instruction with AVX2/AVX-512.

**ISA levels and approach:**

| ISA level | Technique | Throughput |
|-----------|-----------|-----------|
| POPCNT (baseline) | 8-byte chunks: `a64 ^ b64` → `__builtin_popcountll`; 4 independent accumulators to hide latency | ~4–6 GB/s |
| AVX2 | `_mm256_xor_si256` (32 bytes/iter) + bit-sliced Harley-Seal popcount or 4-bit lookup table | ~20–30 GB/s |
| AVX-512VPOPCNTDQ | `_mm512_xor_si512` + `_mm512_popcnt_epi8` (64 bytes/iter); reduce with `_mm512_reduce_add_epi64` after widening | ~40–60 GB/s |

**Dispatch guards:**
```c
/* CPUID: AVX512VPOPCNTDQ */
if (__builtin_cpu_supports("avx512vpopcntdq")) → AVX-512 path
/* CPUID: AVX2 */
else if (__builtin_cpu_supports("avx2"))        → AVX2 path
/* CPUID: POPCNT */
else if (__builtin_cpu_supports("popcnt"))      → POPCNT path
else                                            → scalar fallback
```

**Key implementation notes:**
- All paths need a scalar tail for `n % vector_width` remaining bytes.
- For AVX2 bit-sliced popcount, the Harley-Seal algorithm (see Muła, Kurz,
  Lemire 2017) avoids the latency of a per-byte lookup by operating on 256-bit
  words; it is the standard approach for this ISA level.
- `__builtin_popcountll` compiles to a single `popcnt` instruction with `-mpopcnt`
  or `-march=native`; do not implement popcount manually.
- Accumulate partial 64-bit sums to avoid overflow when processing large arrays.
