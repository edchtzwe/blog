+++
title = '1. From Video to Meaning: Why I Built a Semantic Search Engine'
date = '2026-05-31'
draft = true
tags = ['semantic-search', 'video', 'nlp']
series = ['Semantic Video Search']
+++

## The Problem

- Video is the fastest-growing data type, and the hardest to search
- You can search titles and descriptions, but not what's *inside* the video
- Traditional approach: manual tagging, timestamps, metadata entry. Doesn't scale.
- The gap: AI can analyse video, but most pipelines stop at object detection

## Why Semantic Search?

- Users don't want "show me videos tagged 'cooking'"
- They want "show me the part where they talk about fermentation in sourdough"
- That requires understanding meaning, not matching keywords

## What I Set Out to Build

- Full ingestion pipeline: raw video → structured metadata
- Semantic search: natural language query → relevant video segments
- Constraints: flash lite model, <5 second response, >95% accuracy
- Built solo, AI-assisted toolchain

## What This Series Covers

[Roadmap of the 7 posts]

---

**Status:** Outline.
