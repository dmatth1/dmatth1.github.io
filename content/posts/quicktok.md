---
title: "quicktok: a faster tokenizer"
date: 2026-06-11
description: "quicktok is an exact BPE tokenizer for OpenAI and open-model encodings 2–3.5× faster than the fastest alternative."
tags: ["tokenizer", "performance", "openai"]
ShowToc: true
TocOpen: false
draft: false
---

**quicktok** is a fast, exact BPE tokenizer written in C++. Token ids are byte-identical to `tiktoken`, and encoding runs **2–3.5×** faster than `bpe-openai` (the fastest alternative I know of) and **4–11×** faster than `tiktoken` itself. I believe it's the fastest exact CPU tokenizer available today for these encodings. It ships cl100k, o200k, GPT-OSS (o200k_harmony), Llama-3, and Qwen2.5/3, all byte-exact, plus bring-your-own Llama-4.

This is useful for anyone doing large amounts of CPU-bound data processing — search indexing, ingesting corpora, token counting/billing — and can significantly reduce the time and cost of data ingestion. It can also be used for online request serving, such as CPU-bound inference paths (token counting, embedding serving).

I'm releasing it as a Python library (`pip install quicktok-v1`) and it's available via C++ source. Repo: https://github.com/dmatth1/quicktok.

## Benchmarks

Measured on 3 public corpora on my Apple M1, single thread, MB/s. Every encoder's output was verified token-for-token identical against `tiktoken` before timing.

**cl100k_base** (GPT-3.5 / GPT-4)

  | encoder | The Pile | GitHub code | Common Crawl |
  |---|---:|---:|---:|
  | **quicktok** | **116.1** | **144.2** | **75.2** |
  | bpe-openai | 36.5 | 41.6 | 29.2 |
  | tiktoken-rs | 15.3 | 14.3 | 13.5 |
  | tiktoken (Python) | 14.7 | 13.2 | 12.3 |
  | TokenDagger | 11.5 | 12.0 | 11.2 |

**o200k_base** (GPT-4o)

  | encoder | The Pile | GitHub code | Common Crawl |
  |---|---:|---:|---:|
  | **quicktok** | **100.6** | **117.1** | **59.2** |
  | bpe-openai | 36.1 | 40.1 | 29.9 |
  | tiktoken-rs | 23.1 | 20.9 | 17.9 |
  | tiktoken (Python) | 21.6 | 19.3 | 16.3 |
  | TokenDagger | 11.0 | 11.7 | 10.2 |

**quicktok** also beats llama.cpp's tokenizer on the Llama-3 vocab by **~14×**. The parallel `encode_batch` reaches 706 MB/s native on 8 cores; from Python it sustains 550 MB/s — **24×** `tiktoken`'s batch API. The speedups hold on other architectures like x86.

To keep the comparison fair, each encoder is called through the same raw API its own benchmark uses. (TokenDagger's README claims 2–4× over tiktoken, but that's on Llama-4/Mistral vocabs on AMD EPYC; on cl100k/o200k it lands around Python tiktoken's level.) To reproduce run `make bench-compare` in the repo.

## How it works
  
The fundamental algorithm is the same as `bpe-openai` (exact backtracking BPE)  - see [their blog post](https://github.blog/ai-and-ml/llms/so-many-tokens-so-little-time-introducing-a-faster-more-flexible-byte-pair-tokenizer/). Much of the speedup over `bpe-openai` comes from data-structure engineering around reducing memory accesses.

* **2-byte trie:** the longest-match walk reads 2 input bytes per single 8-byte slot load, with a zero-lookup direct table for CJK (Chinese/Japanese/Korean) characters.
* **Dense validity memos:** merge-validity checks hit exactly-keyed caches (2 MB for 17-bit token ids, a second wide one for 200k-vocab ids; a bijective mixer means no aliasing).
* **Specialized pretokenizer:** the fixed regex is compiled by hand into a scanner, no general regex.

## Caveats
All comparisons are single-threaded by design - parallel/batch is available but single-threaded is fair for comparison. Multilingual text (Common Crawl) is definitely the weakest ratio. Numbers above are from an M1 and were cross-checked on x86 Xeon - the ordering holds on both but absolute MB/s moves with corpus and host. Table numbers show the native C++ runtime of **quicktok** but with using via PyPi numbers are slower - about 1.5x faster than `bpe-openai`.
  
## Closing notes

The full methodology - corpus fetching, the exactness gate, raw-API rules — is in [bench/README.md](https://github.com/dmatth1/quicktok/blob/main/bench/README.md). If you find an input where quicktok's ids differ from tiktoken's that's definitely a bug and please report it!

*Written by Dan Mattheiss*

