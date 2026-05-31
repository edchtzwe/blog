+++
title = '4. Flash Lite, Full Power: Running Semantic Search on a Toaster'
date = '2026-05-31'
draft = true
tags = ['semantic-search', 'optimization', 'models']
categories = ["NLP", "Video Search"]
series = ["Semantic Video Search"]
+++

## The Constraint

- Flash lite model. Minimal VRAM. No GPU cluster.
- Must run semantic search, not just keyword matching.
- Target: production-grade accuracy on hardware that costs less than lunch.

## Model Selection

- [Why a flash lite model and not something bigger]
- [The candidates I evaluated]
- [What I chose and why]

## Making It Work

- [Quantisation: how low can you go before accuracy drops?]
- [Prompt engineering for search: getting the model to understand intent]
- [Caching strategies: compute once, reuse infinitely]

## The Hybrid Approach

- [Flash lite for the heavy lifting, but what's actually heavy?]
- [Pre-computed embeddings + lightweight reranking]
- [Why this combination gave me the best of both worlds]

## Benchmarks

- [Model A vs. Model B on the same dataset]
- [Accuracy vs. latency tradeoff curve]
- [Where the flash lite model actually beat the big ones]

---

**Status:** Outline.
