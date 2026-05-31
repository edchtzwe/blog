+++
title = "6. Under 5 Seconds: Latency Optimisation Tricks They Don't Teach You"
date = '2026-05-31'
draft = true
tags = ['semantic-search', 'performance', 'optimization']
categories = ["NLP", "Video Search"]
series = ["Semantic Video Search"]
+++

## The 5-Second Budget

- [Where does the time actually go? A latency breakdown]
- [The hierarchy: network < disk < compute < model inference]
- [What you can parallelise and what you can't]

## Pre-computation Is Free Latency

- [What to compute before the query arrives]
- [Embedding pre-computation: the single biggest win]
- [Metadata indexing: make lookups O(1) or don't bother]

## Query-Time Optimisations

- [Two-stage retrieval: coarse filter → fine rerank]
- [Result limiting: how many candidates before diminishing returns?]
- [Early termination: when "good enough" beats "perfect"]

## Infrastructure Tricks

- [In-memory everything]
- [Batching without blocking]
- [The async pipeline: fire, forget, collect]

## The Latency Budget (Real Numbers)

| Stage | Before | After |
|---|---|---|
| [Stage 1] | [—ms] | [—ms] |
| [Stage 2] | [—ms] | [—ms] |
| **Total** | **[—]s** | **<5s** |

---

**Status:** Outline.
