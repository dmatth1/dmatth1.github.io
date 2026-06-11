---
title: "quicktok: a faster tokenizer"
date: 2026-06-10
description: "quicktok is an OpenAI-compatible tokenizer that beats all existing tokenizer by 3-10x."
tags: ["tokenizer", "performance", "openai"]
ShowToc: true
TocOpen: false
draft: false
---

arrow-go shipped AVX2/SSE4/NEON SBBF probes in 18.3.0
([PR #336](https://github.com/apache/arrow-go/pull/336)). arrow-cpp
and Velox ship the same scalar reference line-for-line; arrow-rs is
the same algorithm in Rust. Those three cover the C-family Parquet
ecosystem: DuckDB, ClickHouse, Polars, DataFusion, Trino, Presto,
Spark via Gluten, StarRocks, Doris, pyarrow → pandas, Apache Drill.
