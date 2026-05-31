+++
title = 'From Video to Meaning: Why I Built a Semantic Search Engine'
date = '2026-05-31'
draft = false
tags = ['semantic-search', 'video', 'nlp', 'ai']
series = ['Semantic Video Search']
+++

Video is the fastest-growing data type on the internet, and the dumbest.

You can search the title. You can search the description. You can maybe search auto-generated captions if the platform bothered. But you cannot search what's *inside* the video the way you search text.

Try this: find me the exact moment in a 38-minute technical talk where the speaker explains why they chose Rust over Go. Google can't do it. YouTube's search will give you the whole video and hope you scrub through it yourself. If you're lucky, someone left a timestamp in the comments.

That gap — between what video contains and what we can actually search — is what I set out to close.

## The Problem in One Sentence

Video files are sealed objects. Once rendered, they become opaque blocks of pixels and audio. Platforms can compress them, stream them, and run object detection on them. But they cannot easily answer the question: *"Show me the part where they discuss X."*

I wanted to build a system where you could type a natural language query — something like *"find where they talk about fermentation techniques in sourdough"* — and get back the exact segment, not the whole video. No manual tagging. No human metadata entry. Fully automated, from raw video to searchable index.

## What I Built

A semantic video search pipeline. Raw video goes in. Structured, searchable metadata comes out. The pipeline handles:

1. **Ingestion** — frame extraction, transcription, and NLP processing that turns hours of video into structured data
2. **Embedding** — converting video segments into a vector space where semantic similarity maps to actual meaning
3. **Search** — natural language queries that retrieve relevant segments with >95% accuracy in under 5 seconds

And I built it to run on a flash lite model. Not a GPU cluster. Not a cloud of TPUs. The kind of hardware you could feasibly run on a decent laptop.

## The Hard Parts

There are three problems that make video search genuinely difficult, and they compound each other.

**First, video is multimodal.** There's the visual track (what's on screen), the audio track (what's being said), and the structural layer (scene changes, pacing, chapter boundaries). Any one of these is noisy on its own. Putting them together in a way that produces reliable search results is a coordination problem before it's a machine learning problem.

**Second, transcripts are not enough.** A raw transcript gives you words, but words without context fail on semantic queries. "How to make a roux" and "the history of French sauces" might share vocabulary but answer completely different questions. You need to understand what each segment is *about*, not just what words it contains.

**Third, speed and accuracy pull in opposite directions.** The brute-force approach — transcribe everything, embed everything, run a large model over every possible segment — works fine. It's also slow and expensive. Getting the same quality on a flash lite model, in under five seconds, requires being clever about what you compute and when.

## The Constraints I Set

I gave myself specific constraints because constraints force design decisions:

| Constraint | Why It Matters |
|---|---|
| Flash lite model | If it requires a data center, it's a demo, not a product |
| <5 second response | Search that takes longer than a page load might as well not exist |
| >95% accuracy | Precision without recall is useless; recall without precision is noise |
| Solo developer | Nobody to delegate to. Every decision had to earn its complexity |
| AI-assisted toolchain | I used AI tools heavily — not as a crutch, but as a multiplier |

These constraints shaped every architecture decision that followed. I'll unpack each one in detail over the next posts.

## What This Series Covers

I'm going to walk through the full system, end to end. Not the polished retrospective where everything worked perfectly, but the actual build — the decisions, the dead ends, and the things I'd do differently.

| Part | Topic |
|---|---|
| [This post](#) | The problem, the constraints, and why semantic video search matters |
| Part 2 | The ingestion pipeline: turning raw video into structured metadata |
| Part 3 | Embeddings that actually work: vector strategies for video content |
| Part 4 | Flash lite, full power: running production search on minimal hardware |
| Part 5 | The accuracy play: ground truth, validation, and the metrics that matter |
| Part 6 | Under 5 seconds: latency optimisation tricks |
| Part 7 | The full architecture: what I got right, what I got wrong, and what I'd rebuild |

The target was 6-minute videos — trailers, corporate presentations, cooking shows — the kind of content you find on YouTube. But I stress-tested the pipeline on longer material too. The longest single video I ran through end to end was 38 minutes. It held up.

## Who This Is For

If you're an engineer building search, working with video, or just curious about what's possible when you combine modern NLP tooling with some creative architecture — this is for you. If you're an investor or executive trying to understand why "AI video search" isn't just another buzzword, this is also for you. The CIO presented an early version of this system to an audience of investors and insiders. The reaction told me the interest is real.

I'm not going to name companies or reveal anything proprietary. This is about architecture, technique, and lessons learned. You should be able to read this series and walk away with enough to build something similar.

Next up: the ingestion pipeline. How do you take hours of raw video and turn it into something a search engine can actually use?
