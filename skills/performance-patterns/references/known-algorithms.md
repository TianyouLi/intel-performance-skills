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
| Cosine Similarity | `cosine_similarity`, `cosine_sim`, `cos_sim`, `cosine_distance`, `angular_similarity`, `dot_normalized` |
| Hamming Distance | `hamming_distance`, `hamming_dist`, `hamming`, `count_differing_bits`, `bit_diff_count`, `popcount_xor` |

---

## Cosine Similarity

**What it computes:** The cosine of the angle between two vectors:
`cos(θ) = dot(A, B) / (|A| * |B|)`, where `dot(A,B) = Σ a[i]*b[i]` and
`|A| = sqrt(Σ a[i]²)`. Returns 1.0 for identical direction, 0.0 for
orthogonal, −1.0 for opposite. Widely used in ML embeddings, NLP, and
recommendation systems.

**Why scalar is slow:** Three serial accumulator loops (dot product, two norms)
each have loop-carried FP dependencies. A naive implementation makes three
passes over the data; a smart one makes a single pass but still serializes
on one accumulator. Modern CPUs support FMA (fused multiply-add) and can
process 8–16 floats per cycle with AVX2/AVX-512.

**Key insight — single-pass, multi-accumulator:** Compute `dot_ab`, `dot_aa`,
and `dot_bb` in one loop with independent SIMD accumulators for each. Combine
at the end: `result = dot_ab / sqrt(dot_aa * dot_bb)`. This reads each array
once and exploits instruction-level parallelism across the three FMA streams.

**ISA levels and approach:**

| ISA level | Technique |
|-----------|-----------|
| FMA (baseline) | 4 independent `float` accumulators per stream; `fmaf(a[i], b[i], dot_ab)` etc.; combine with `sqrtf` |
| AVX2 + FMA | `_mm256_fmadd_ps` (8 floats/iter), 4 accumulators per stream (12 YMM registers total); horizontal reduce with `_mm256_hadd_ps` at the end |
| AVX-512 + FMA | `_mm512_fmadd_ps` (16 floats/iter), 4 accumulators per stream; reduce with `_mm512_reduce_add_ps` |

**Dispatch guards:**
```c
/* CPUID: AVX512F */
if (__builtin_cpu_supports("avx512f"))  → AVX-512 path
/* CPUID: AVX2,FMA */
else if (__builtin_cpu_supports("avx2") &&
         __builtin_cpu_supports("fma")) → AVX2+FMA path
/* CPUID: FMA */
else if (__builtin_cpu_supports("fma")) → scalar FMA path
else                                    → scalar fallback
```

**Key implementation notes:**
- Pre-normalized inputs (`|A| = |B| = 1`) reduce to a plain dot product —
  detect this case and skip the norm computation.
- Guard against division by zero: if `dot_aa * dot_bb < epsilon`, return 0.
- FP associativity: multi-accumulator results may differ from a serial sum by
  rounding ε; document this at the function boundary.
- All paths need a scalar tail for `n % vector_width` remaining elements.
- The final `sqrt` and division are a negligible fraction of runtime for
  any array longer than ~16 elements; optimize the loop, not the epilogue.

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
