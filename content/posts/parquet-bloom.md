---
title: "The entire Parquet ecosystem ships a scalar bloom filter probe"
date: 2026-05-15
description: "Apache Arrow C++, arrow-rs, and Velox all ship line-for-line identical scalar SBBF probes. Here's the AVX2 drop-in that's bit-identical and 3–5× faster."
tags: ["bloom-filter", "performance", "parquet", "simd", "apache-arrow"]
ShowToc: true
TocOpen: false
draft: false
---

The Parquet Split-Block Bloom Filter (SBBF) was designed in 2018 around
256-bit blocks — exactly the size of one `__m256i` AVX2 register. Every
on-disk constant in the spec is laid out to make AVX2 the natural
implementation.

Eight years later, no native Parquet reader I could find uses AVX2 in
the probe path.

Apache Arrow C++, arrow-rs, and Meta's Velox all ship the same scalar
8-iteration loop with an early-exit branch. The pattern is line-for-line
identical across all three. Together those three readers cover
essentially the entire native Parquet ecosystem: DuckDB, ClickHouse,
Polars, DataFusion, Trino, Presto, Spark-native, StarRocks, Doris,
pyarrow → pandas, Apache Drill.

This post walks through what's there, why it's slower than it needs to
be, and the bit-identical AVX2 drop-in that fixes it.

## The evidence: three line-for-line identical scalar implementations

**Apache Arrow C++** ([`apache/arrow/cpp/src/parquet/bloom_filter.cc:348`](https://github.com/apache/arrow/blob/main/cpp/src/parquet/bloom_filter.cc#L348)),
the reference implementation forked by most C++ Parquet readers:

```cpp
bool BlockSplitBloomFilter::FindHash(uint64_t hash) const {
    const uint32_t bucket_index = static_cast<uint32_t>(
        ((hash >> 32) * (num_bytes_ / kBytesPerFilterBlock)) >> 32);
    const uint32_t key = static_cast<uint32_t>(hash);
    const uint32_t* bitset32 = reinterpret_cast<const uint32_t*>(data_->data());

    for (int i = 0; i < kBitsSetPerBlock; ++i) {
        const uint32_t mask = UINT32_C(0x1) << ((key * SALT[i]) >> 27);
        if (ARROW_PREDICT_FALSE(0 ==
                (bitset32[kBitsSetPerBlock * bucket_index + i] & mask))) {
            return false;
        }
    }
    return true;
}
```

**arrow-rs** ([`apache/arrow-rs/parquet/src/bloom_filter/mod.rs:241`](https://github.com/apache/arrow-rs/blob/main/parquet/src/bloom_filter/mod.rs#L241)),
used by Polars and DataFusion:

```rust
fn check(&self, hash: u32) -> bool {
    let mask = Self::mask(hash);
    for i in 0..8 {
        if self[i] & mask[i] == 0 {
            return false;
        }
    }
    true
}
```

**Velox** ([`facebookincubator/velox/velox/dwio/parquet/common/BloomFilter.cpp:196`](https://github.com/facebookincubator/velox/blob/main/velox/dwio/parquet/common/BloomFilter.cpp#L196)),
Meta's C++ execution engine that powers Prestissimo / Trino / native Spark:

```cpp
bool BlockSplitBloomFilter::findHash(uint64_t hash) const {
    const uint32_t bucketIndex = static_cast<uint32_t>(
        ((hash >> 32) * (numBytes_ / kBytesPerFilterBlock)) >> 32);
    const uint32_t key = static_cast<uint32_t>(hash);
    const uint32_t* bitset32 =
        reinterpret_cast<const uint32_t*>(data_->as<char>());

    for (int i = 0; i < kBitsSetPerBlock; ++i) {
        const uint32_t mask = UINT32_C(0x1) << ((key * SALT[i]) >> 27);
        if (0 == (bitset32[kBitsSetPerBlock * bucketIndex + i] & mask)) {
            return false;
        }
    }
    return true;
}
```

Eight dependent 32-bit loads per probe, an early-exit branch after each.
No SIMD. The compilers don't auto-vectorize this either — the early
`return false` prevents LLVM and GCC from proving equivalence with a
horizontal AND-test across the whole 256-bit block.

## Why this is surprising

The Parquet SBBF spec from 2018 dictates these exact constants:

- **256-bit blocks** = one `__m256i` register
- **8 bits set per block** = 8 lanes of a 32-bit-lane AVX2 register
- **Bit position formula** `(key * SALT[i]) >> 27` for `i = 0..7` =
  `vpmulld + vpsrld` across 8 lanes
- **"All 8 bits set?" check** = `_mm256_testc_si256` in one op

The on-disk format is, in essence, an instruction sheet for an AVX2
implementation. The spec even provides the eight specific 32-bit SALT
constants, which directly slot into `_mm256_load_si256(SALT)`.

Yet the reference C++ implementation in Apache Arrow was scalar from
day one, the Rust port in arrow-rs was line-for-line ported from the
C++, and Velox copied the same scalar shape from arrow-cpp. The pattern
propagated because each new reader took the reference as authoritative.

## The AVX2 fast path

Same on-disk format. Same SALT constants. Same bit positions. **Same
output for every input.** Only the probe-implementation changes:

```cpp
bool find_avx2(uint64_t hash) const {
    uint32_t bucket_index = uint32_t(((hash >> 32) * num_blocks_) >> 32);
    uint32_t key = uint32_t(hash);

    // Compute 8 mask bits in parallel.
    // Each lane: 1 << ((key * SALT[i]) >> 27)
    __m256i hash_v = _mm256_set1_epi32(int32_t(key));
    __m256i salt   = _mm256_load_si256(reinterpret_cast<const __m256i*>(SALT));
    __m256i prod   = _mm256_mullo_epi32(hash_v, salt);
    __m256i shift  = _mm256_srli_epi32(prod, 27);
    __m256i ones   = _mm256_set1_epi32(1);
    __m256i mask   = _mm256_sllv_epi32(ones, shift);

    // Load the 256-bit block (one cache line).
    const __m256i* blk = reinterpret_cast<const __m256i*>(
        data_ + bucket_index * 32);
    __m256i blk_v = _mm256_loadu_si256(blk);

    // testc returns 1 iff (~blk_v & mask) == 0
    //   <=> every bit in mask is also set in blk_v
    return _mm256_testc_si256(blk_v, mask);
}
```

About a dozen instructions, no branches. Compiles to roughly seven
cycles in the hot path.

## The bit-identical guarantee

I built a standalone bench harness that runs the scalar and AVX2 probe
paths side-by-side on the same Parquet-format data — same SALT
constants, same XXH64 hash, same Lemire fastrange bucket index — and
compared their outputs on every query.

**0 mismatches across 167 million (query, filter) pairs.**

The math is straightforward: `_mm256_mullo_epi32(hash_v, salt)` produces
eight 32-bit lane-wise products, each equal to the scalar
`key * SALT[i]`. The subsequent shifts produce the same 8 bit positions.
`_mm256_testc_si256` is the SIMD equivalent of "all mask bits set in
block." Logically identical to the scalar loop, just done in parallel.

This is what makes the change a particularly low-friction PR: reviewers
don't have to convince themselves the AVX2 path is correct under edge
cases. The bit-identical property means it produces the same output as
the existing scalar reference on every input.

## The numbers

Measured on a virtualised Intel Xeon @ 2.8 GHz (Cascade Lake-class,
33 MiB L3), `g++ -O3 -mavx2 -mbmi2`:

| Regime | Total filter footprint | scalar | AVX2 single-key | AVX2 4-way bulk |
|---|---:|---:|---:|---:|
| Small (in-cache) | 0.5 MiB | 12.73 ns | **3.70 ns (3.4×)** | **2.36 ns (5.4×)** |
| Medium (out-of-L3) | 128 MiB | 35.84 ns | 29.18 ns (1.2×) | **22.79 ns (1.6×)** |
| Large (deep DRAM) | 1 GiB | 41.41 ns | 35.90 ns (1.15×) | **27.75 ns (1.5×)** |

In-cache, the AVX2 path is **3.4× faster** because the bottleneck is
ALU work — the scalar 8-iteration loop has 8 dependent loads with
branches; the AVX2 path is two SIMD ops after one cache-line load.

Out-of-L3, the per-probe gain narrows to ~1.15–1.2× because both
paths now wait for the same single DRAM load. The Parquet SBBF was
already cache-line-blocked since 2018 — that win was cashed long ago.
The remaining gain is just ALU work, which is small compared to memory
latency.

The 4-way bulk-probe path (probing one query hash against four
row-group filters in parallel) recovers most of the out-of-L3 gap by
overlapping four cache misses in flight simultaneously. That's worth
~1.5× out-of-L3 — more than triple the single-key out-of-L3 gain.

## What this does and doesn't mitigate

The Parquet ecosystem moved through a "should we displace bloom?" cycle
recently — Databricks deprecated Parquet bloom in Delta in favor of
Photon predictive I/O + liquid clustering. That displacement was driven
by storage overhead and maintenance cost, not by probe CPU.

| Concern | AVX2 fix mitigates? |
|---|---|
| Probe CPU on large queries | **Yes — 3–5× in-cache, 1.15–1.5× out-of-L3** |
| Build CPU during writes | Yes with 4-way `bulk_add` |
| `WHERE col IN (val1, val2, ...)` cost | Yes — 4-way bulk amortises across query values |
| Storage overhead (~10 bits/value extra per file) | No |
| FP rate vs alternatives (min/max + page indexes) | No |
| Maintenance cost on UPDATE/MERGE rebuild | Partial |

This doesn't argue against Photon-style displacement on storage
grounds. What it does is **widen the envelope of queries where keeping
the bloom beats alternative skipping mechanisms**. With the AVX2 path,
the "is bloom worth it for this workload?" threshold moves up in
row-group-count terms.

For ecosystems that don't have Photon-style alternatives — Iceberg,
Hudi, the entire OSS lakehouse, native engines like DuckDB and
ClickHouse — this is a direct improvement to the core scan path with no
behavior change.

## At scale

Take the 1 GiB-filter-footprint regime: ~224 µs CPU saved per
`SELECT WHERE col = 'X'` query (4-way bulk vs scalar). Honest scaling:

| Query volume | CPU saved per day | What it actually is |
|---|---|---|
| 100k queries/day | 22 sec | invisible |
| 10M queries/day | 37 min | ~0.25 core continuously |
| 100M queries/day | ~6 hours | ~16 cores continuously |
| 1B queries/day | ~62 hours | ~160 cores continuously |

Per-query the savings is invisible. At cloud-lakehouse query volumes
it's meaningful but not transformative — bloom is 2–5% of total query
CPU in most realistic engines, so a 1.5× speedup on that slice is
~1–3% on total wall time.

The case for the patch isn't any single deployment's gain. It's that
**one patch in Apache Arrow C++ ships to ~20 downstream engines and
runs for 5+ years.** The total CPU-years saved is the integral, not
the slice.

## What about ARM?

This is x86 AVX2 only. ARM is roughly 15–25% of cloud server volume
in 2026 and growing (Graviton 3/4, Cobalt 100, Apple Silicon). A
NEON port is straightforward — the intrinsics map almost line-for-line,
with two ops per 256-bit block since NEON is 128-bit. SVE2 fits the
8-lane pattern naturally and would run on one 256-bit register on
Graviton 3/4.

For the upstream Arrow patch, the AVX2 path lands behind a CpuInfo
runtime dispatch; non-AVX2 builds (ARM, RISC-V, older x86) keep the
scalar path unchanged. NEON / SVE2 are natural follow-up patches.

## The repo

Standalone bench with the diff-test guarantee, vendored scalar
references from arrow-cpp / arrow-rs / Velox for diffing against
upstream, and full results:

**[github.com/dmatth1/quickbloom](https://github.com/dmatth1/quickbloom)**

The relevant directories (all under `investigations/`):

- `investigations/parquet_port/` — the bench. `make && ./bench`
  reproduces the numbers above. Diff-test runs every time.
- `investigations/parquet_port/reference/` — vendored Apache Arrow C++ /
  arrow-rs / Velox source so future contributors can diff against
  current upstream.
- `investigations/PARQUET.md` — the technical analysis.

## What I'd like to see

The AVX2 path is small enough that a maintainer could pick it up
directly. The Apache Arrow C++ integration pattern would be a new
translation unit `cpp/src/parquet/bloom_filter_avx2.cc` behind
`CpuInfo::GetInstance()->IsSupported(CpuInfo::AVX2)`, matching the
existing pattern in `arrow/util/bit_util_avx2.cc` and
`arrow/util/hashing_avx2.cc`.

arrow-rs would be a `target_feature = "avx2"` function in
`parquet/src/bloom_filter/mod.rs` with `is_x86_feature_detected!`
runtime dispatch. Velox would follow `velox/dwio/parquet/common/`.

The bit-identical diff test makes review almost mechanical. If any
maintainer reading this wants to take the patch under their own
attribution, the code is ready to drop in — see the linked repo.

If not, I'll send a `[DISCUSS]` to `dev@arrow.apache.org` and work
through the standard process.

## Acknowledgments

This is a small slice of a larger body of work on single-key Bloom
filter performance (a differential-evolution harness for systems code,
separate v11/v14 ports for Scylla, DuckDB, Reth, RedisBloom). The
Parquet finding came out of mapping what's actually shipped vs what's
measured in the literature. Surprised it was sitting there.
