---
title: "quicktok: a faster tokenizer"
date: 2026-06-10
description: "quicktok is an OpenAI-compatible tokenizer that beats all existing tokenizers by 3-10x."
tags: ["tokenizer", "performance", "openai"]
ShowToc: true
TocOpen: false
draft: false
---

**quicktok** is a fast, exact BPE tokenizer for OpenAI encodings written in C++. Measured at **6–8×** faster than `tiktoken` and **3x** faster than `bpe-openai`,  I believe it's the fastest tokenizer available today. This is useful for anyone doing large amounts of data processing and indexing (search indexing, ingesting corpora) and can significantly reduce time and costs of data ingestion. It can also be used for online request serving such as for inference. 

I'm releasing it as a Python library `pip install quicktok` and it's available via C++ source. Repo is publicly available at https://github.com/dmatth1/quicktok.

## Benchmarks

Measured on the same corpus on my Apple M1 and single-thread, MB/s:

**cl100k_base** (GPT-3.5 / GPT-4)

| encoder | The Pile | Code | Common Crawl |
|---|---:|---:|---:|
| **quicktok** | **116.1** | **144.2** | **75.2** |
| bpe-openai | 36.5 | 41.6 | 29.2 |
| tiktoken-rs | 15.3 | 14.3 | 13.5 |
| tiktoken (Python) | 14.7 | 13.2 | 12.3 |
| TokenDagger | 11.5 | 12.0 | 11.2 |

**o200k_base** (GPT-4o)

| encoder | The Pile | Code | Common Crawl |
|---|---:|---:|---:|
| **quicktok** | **100.6** | **117.1** | **59.2** |
| bpe-openai | 36.1 | 40.1 | 29.9 |
| tiktoken-rs | 23.1 | 20.9 | 17.9 |
| tiktoken (Python) | 21.6 | 19.3 | 16.3 |
| TokenDagger | 11.0 | 11.7 | 10.2 |

**quicktok** also beats llama.cpp's tokenizer on the Llama-3 vocab by **~14x**. The `encode_batch` function runs tokenization in parallel and achieves speeds up to 706 MB/s (**24×** faster than tiktoken batch) on my M1 macbook. The speedups hold on other architectures like x86.

## How it works

The fundamental algorithm is the same as `bpe-openai`(exact backtracking BPE) - see [the blog post](https://github.blog/ai-and-ml/llms/so-many-tokens-so-little-time-introducing-a-faster-more-flexible-byte-pair-tokenizer/). Much of the speedup over `bpe-openai` comes from data-structure engineering:

* 2-byte trie: the longest-match walk reads 2 input bytes per single 16-byte cache load, with a zero-lookup direct table for CJK characters.
* Dense validity memo: merge-validity checks hit a 2 MB exactly-keyed cache (a bijective mixer means no aliasing, ever).
* Specialized pretokenizer: the fixed regex is compiled by hand into a scanner; no general regex engine anywhere.

## Closing notes


