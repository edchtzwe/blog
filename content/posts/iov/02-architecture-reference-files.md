+++
title = '2. The Architecture: Reference Files, Content Routing, and the Control Plane'
date = '2026-05-30'
draft = true
tags = ['architecture', 'systems', 'video']
series = ['Video Virtualisation']
+++

## Core Components

### Reference File Format
- [Describe the structure: player control parameters, linking data, timing/sync info]
- [Why this format vs. existing standards (SMIL, MPEG-DASH manifests)]
- [Size considerations: reference file must be tiny relative to content]

### Content Source Routing
- [How the player locates and retrieves content from multiple sources]
- [Multi-source sync: clock drift, buffer strategy, failure handling]
- [Comparison to HLS/DASH segment-based approaches]

### The Control Plane (Compiler)
- [How the reference file compiler works]
- [Request → identify sources → generate reference file → serve]
- [On-the-fly vs. pre-computed reference files]

### Security & Access Control
- [How content sources authenticate the player]
- [DRM considerations — or why we didn't bother]

## Design Trade-offs

| Decision | Chose | Rejected | Why |
|---|---|---|---|
| [Format] | [—] | [—] | [—] |
| [Caching] | [—] | [—] | [—] |
| [Sync strategy] | [—] | [—] | [—] |

---

**Status:** Outline. Fill the architecture table and decisions. Diagrams welcome.
