+++
title = '1. How I Built a Video Virtualisation Engine From Scratch'
date = '2026-05-30'
draft = true
tags = ['architecture', 'video', 'systems']
series = ['Video Virtualisation']
+++

## Problem

- Video files are sealed objects — rendered, static, impossible to recombine without re-rendering
- AI can analyse video but can't restructure it dynamically
- The industry treats video as a finished asset, not a programmable medium

## What I Built

- A system that separates video structure (reference files) from video data (content sources)
- Reference file = playlist on steroids: timing, sequencing, source routing, player control params
- Player reads reference file, pulls chunks from multiple sources, plays as seamless stream
- Reference file compiler that generates these on the fly

## Key Technical Decisions

- [Detail the architecture: control plane, content sources, reference file format]
- [Why this approach vs. traditional transcoding pipelines]
- [Trade-offs: latency, cache strategy, failure modes]

## Result

- Working prototype. Built solo. AI-assisted toolchain.
- Lessons from shipping something this complex with a tiny team.

---

**Status:** Outline. Fill with architecture details, decisions, and lessons. No company names.
