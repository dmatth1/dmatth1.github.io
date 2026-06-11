---
title: "quicktok: a faster tokenizer"
date: 2026-06-10
description: "quicktok is an OpenAI-compatible tokenizer that beats all existing tokenizer by 3-10x."
tags: ["tokenizer", "performance", "openai"]
ShowToc: true
TocOpen: false
draft: false
---

**quicktok** is a fast, exact BPE tokenizer for OpenAI encodings written in C++. Measured at 6–8× faster than `tiktoken` and 3x faster than `bpe-openai`,  I believe it's the faster tokenizer available. This is useful for anyone doing large amounts of data processing and indexing (search indexing, ingesting corpora) and can significantly reduce time and costs of data ingestion. It can also be used for online transactions processing such as for inference.

## Benchmarks

Measured on the same corpus on my Apple M1 and single-thread:

| encoder | The Pile (40 MB) | code (20 MB) |
|---|---:|---:|
| **quicktok** | **112.8 MB/s** | **149.3 MB/s** |
| bpe-openai | 35.9 | 37.7 |
| tiktoken-rs | 14.9 | 12.9 |
| tiktoken (Python) | 14.0 | 12.1 |
| TokenDagger | 10.8 | 12.2 |


