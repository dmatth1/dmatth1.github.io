---
title: "quicktok: a faster tokenizer"
date: 2026-06-10
description: "quicktok is an OpenAI-compatible tokenizer that beats all existing tokenizer by 3-10x."
tags: ["tokenizer", "performance", "openai"]
ShowToc: true
TocOpen: false
draft: false
---

Github link: https://github.com/dmatth1/quicktok

**quicktok** is a fast, exact BPE tokenizer for OpenAI encodings written in C++. Measured at **6–8×** faster than `tiktoken` and **3x** faster than `bpe-openai`,  I believe it's the faster tokenizer available. This is useful for anyone doing large amounts of data processing and indexing (search indexing, ingesting corpora) and can significantly reduce time and costs of data ingestion. It can also be used for online transactions processing such as for inference. I'm releasing it as a Python library `pip install quicktok` and it's available via C++ source. 

## Benchmarks

Measured on the same corpus on my Apple M1 and single-thread:

| encoder | The Pile (40 MB) | code (20 MB) |
|---|---:|---:|
| **quicktok** | **112.8 MB/s** | **149.3 MB/s** |
| bpe-openai | 35.9 | 37.7 |
| tiktoken-rs | 14.9 | 12.9 |
| tiktoken (Python) | 14.0 | 12.1 |
| TokenDagger | 10.8 | 12.2 |

**quicktok** also beats llama.cpp's tokenizer on the Llama-3 vocab by **~14x**. The `encode_batch` function runs tokenization in parallel and achieves speeds up to 550 MB/s (**24×** faster than tiktoken batch) on my M1 macbook. The speedups hold on other architectures like x86.

## How it works

The fundamental algorithm is the same as `bpe-openai`(exact backtracking BPE) - see [the blog post](https://github.blog/ai-and-ml/llms/so-many-tokens-so-little-time-introducing-a-faster-more-flexible-byte-pair-tokenizer/). Much of the speedup over `bpe-openai` comes from data-structure engineering:

* 2-byte trie: the longest-match walk reads 2 input bytes per single 16-byte cache load, with a zero-lookup direct table for CJK characters.
* Dense validity memo: merge-validity checks hit a 2 MB exactly-keyed cache (a bijective mixer means no aliasing, ever).
* Specialized pretokenizer: the fixed regex is compiled by hand into a scanner; no general regex engine anywhere.

## Closing notes


