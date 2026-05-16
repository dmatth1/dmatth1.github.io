---
title: "arrow-cpp, arrow-rs, and Velox still ship the scalar Parquet bloom probe"
date: 2026-05-15
description: "arrow-go shipped AVX2/SSE4/NEON SBBF probes in 18.3.0 (PR #336). The other three readers still ship the scalar reference. A C++ port, bit-identical and 3–5× faster on the probe microbenchmark."
tags: ["bloom-filter", "performance", "parquet", "simd", "apache-arrow"]
ShowToc: true
TocOpen: false
draft: false
---

arrow-go shipped AVX2/SSE4/NEON SBBF probes in 18.3.0
([PR #336](https://github.com/apache/arrow-go/pull/336)). arrow-cpp,
arrow-rs, and Velox still ship the scalar reference, line-for-line
identical. Those three cover the C-family Parquet ecosystem: DuckDB,
ClickHouse, Polars, DataFusion, Trino, Presto, Spark-native,
StarRocks, Doris, pyarrow → pandas, Apache Drill.

A C++ port of the arrow-go approach: bit-identical on 167M
(query, filter) pairs, 3–5× in-cache on the probe microbenchmark,
1.5× out-of-L3 with a 4-way bulk path across row-group filters.

## The three still-scalar implementations

Apache Arrow C++ ([`bloom_filter.cc:348`](https://github.com/apache/arrow/blob/main/cpp/src/parquet/bloom_filter.cc#L348)),
the reference forked by most C++ readers:

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

arrow-rs ([`mod.rs:241`](https://github.com/apache/arrow-rs/blob/main/parquet/src/bloom_filter/mod.rs#L241)),
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

Velox ([`BloomFilter.cpp:196`](https://github.com/facebookincubator/velox/blob/main/velox/dwio/parquet/common/BloomFilter.cpp#L196)),
which powers Prestissimo, Trino, and Spark-native:

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

Eight dependent 32-bit loads per probe, early-exit branch after each.
LLVM and GCC don't auto-vectorize this — the early `return false`
blocks the equivalence proof with a horizontal AND-test across the
block.

## The spec maps onto AVX2

The 2018 SBBF spec dictates:

- 256-bit blocks = one `__m256i`.
- 8 bits set per block = 8 lanes of a 32-bit AVX2 register.
- Bit position formula `(key * SALT[i]) >> 27` for `i = 0..7` =
  `vpmulld + vpsrld` across 8 lanes.
- "All 8 bits set?" check = `_mm256_testc_si256` in one op.
- The eight SALT constants slot into `_mm256_load_si256(SALT)`.

The on-disk format reads like an instruction sheet for AVX2. Impala
and Kudu shipped AVX2 SBBF probes years ago; arrow-go added
AVX2/SSE4/NEON last year. Apache Arrow C++ shipped the scalar
pseudocode reference from day one, and arrow-rs and Velox ported
from it, scalar shape and all.

## The AVX2 probe

Same on-disk format, same SALT, same bit positions. Only the probe
changes:

```cpp
bool find_avx2(uint64_t hash) const {
    uint32_t bucket_index = uint32_t(((hash >> 32) * num_blocks_) >> 32);
    uint32_t key = uint32_t(hash);

    __m256i hash_v = _mm256_set1_epi32(int32_t(key));
    __m256i salt   = _mm256_load_si256(reinterpret_cast<const __m256i*>(SALT));
    __m256i prod   = _mm256_mullo_epi32(hash_v, salt);
    __m256i shift  = _mm256_srli_epi32(prod, 27);
    __m256i ones   = _mm256_set1_epi32(1);
    __m256i mask   = _mm256_sllv_epi32(ones, shift);

    const __m256i* blk = reinterpret_cast<const __m256i*>(data_ + bucket_index * 32);
    return _mm256_testc_si256(_mm256_loadu_si256(blk), mask);
}
```

About a dozen instructions, no branches, ~7 cycles in the hot path.

`_mm256_mullo_epi32(hash_v, salt)` produces eight 32-bit products,
each equal to scalar `key * SALT[i]`. The shifts produce the same 8
bit positions. `_mm256_testc_si256` returns 1 iff `(~block & mask) == 0`
— the same condition as "every mask bit set in the block."
Bit-identical to the scalar path on every input.

The bench in the repo runs both paths on the same Parquet-format
data (same SALT, same XXH64, same Lemire fastrange bucket index): 0
mismatches across 167M (query, filter) pairs.

## Numbers

Intel Xeon @ 2.8 GHz (Cascade Lake-class, 33 MiB L3, virtualised),
`g++ -O3 -mavx2 -mbmi2`:

| Regime             | Filter footprint |   scalar | AVX2 single-key      | AVX2 4-way bulk      |
|--------------------|-----------------:|---------:|---------------------:|---------------------:|
| Small (in-cache)   |          0.5 MiB | 12.73 ns | **3.70 ns (3.4×)**   | **2.36 ns (5.4×)**   |
| Medium (out-of-L3) |          128 MiB | 35.84 ns | 29.18 ns (1.2×)      | **22.79 ns (1.6×)**  |
| Large (deep DRAM)  |            1 GiB | 41.41 ns | 35.90 ns (1.15×)     | **27.75 ns (1.5×)**  |

In-cache the bottleneck is ALU work: 8 dependent loads with branches
vs. two SIMD ops after one cache-line load. 3.4× is the ALU collapse.

Out-of-L3 both paths wait for the same DRAM load and the per-probe
gain narrows to 1.15–1.2×. The 4-way bulk path (one query hash
probed against four row-group filters in parallel) overlaps four
cache misses in flight and recovers ~1.5×.

## What the AVX2 path touches

Scoped to the probe stage, not query wall time:

| Concern                                          | AVX2 changes it?                                  |
|--------------------------------------------------|---------------------------------------------------|
| Probe-stage CPU                                  | Yes — 3–5× in-cache, 1.15–1.5× out-of-L3 (microbench) |
| `WHERE col IN (val1, val2, ...)` probe cost      | Yes — 4-way bulk amortises across values          |
| Build-stage CPU during writes                    | Yes — bulk insert path                            |
| Storage overhead (~10 bits/value extra per file) | No                                                |
| FP rate vs alternatives (min/max + page indexes) | No                                                |
| Maintenance cost on UPDATE/MERGE rebuild         | No                                                |

Same on-disk format, no behaviour change for any reader.

## ARM

x86 AVX2 only — no NEON or SVE2 measurements. The intrinsics map
almost line-for-line on paper (NEON in two 128-bit ops per block;
SVE2 in one register on Graviton 3/4), but the ratios need measuring.
The ALU collapse should still apply since the dependent-load chain
is the same; rough estimate 2–2.5× in-cache.

Upstream the AVX2 path lands behind a CpuInfo runtime dispatch;
non-AVX2 builds keep the scalar path.

## The repo

Standalone bench, vendored scalar references from arrow-cpp /
arrow-rs / Velox for diffing against upstream, full results:

**[github.com/dmatth1/quickbloom](https://github.com/dmatth1/quickbloom)**

Under `investigations/`:

- `parquet_port/` — the bench. `make && ./bench` reproduces the
  numbers. Diff-test runs every time.
- `parquet_port/reference/` — verbatim source from the three
  upstream readers.
- `PARQUET.md` — technical analysis.
