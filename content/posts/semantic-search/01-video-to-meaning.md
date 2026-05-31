+++
title = 'From Video to Meaning: Why I Built a Semantic Search Engine'
date = '2026-05-31'
draft = false
tags = ['semantic-search', 'video', 'nlp', 'ai']
categories = ['NLP', 'Video Search']
series = ['Semantic Video Search']
+++

Video is the fastest-growing data type on the internet, and the dumbest.

You can search titles and descriptions. Gemini can even find timestamped YouTube links for you — and that's all you get. You click it, it opens on YouTube, you watch the video. End of workflow.

There's no data for you to work with. You can't build anything on top of a link. If the first result is wrong, you're stuck — Gemini won't give you a list of alternatives by default anyway. And if it did, you'd have to watch every single one to find what you need.

With structured metadata, you scan the data yourself and decide if it's worth watching. Data you can build with. Links you just click.

## What I Wanted

I didn't want my video sitting on someone else's platform indefinitely. Some clients have specifications that mandate their content never exists off-prem at all. That's non-negotiable.

So I took a different approach. **Upload, extract, purge.**

My pipeline sends video to Gemini just long enough to pull out metadata — transcription, scene detection, semantic structure. Once the metadata is extracted, the file disappears. Gemini's Files API wipes uploads after 48 hours. Vertex AI goes further with no-retention contracts — they don't keep your data at all.

Not every provider works this way. 12 Labs keeps your video. Which means Marengo does too. Which means Bedrock via Marengo does. Read the fine print before you upload anything proprietary.

With Google Files API, the video never stays for more than 48 hours. You get to keep the metadata, how you store it is up to you. And the metadata is all you need for search — and everything downstream. RAG, MCP tool calls, A2A agent chaining — it all starts with structured data.

No manual tagging. No platform lock-in. Raw files in, structured metadata out.

On top of that, I wanted actual search. A search engine where you type *"where does he explain how Sovereign manages 2,000 Envoy proxies across 13 regions?"* and get the exact segment. Because yes, I use this to study engineering content too. Thanks Vasilios you absolute legend.

## What I Built

A semantic video search pipeline. Raw video goes in. Structured, searchable metadata comes out. The pipeline handles:

1. **Ingestion** — frame extraction, transcription, and NLP processing that turns hours of video into structured data
2. **Embedding** — converting video segments into a vector space where semantic similarity maps to actual meaning
3. **Search** — natural language queries that retrieve relevant segments with >95% accuracy in under 5 seconds

And I built it to run on a flash lite model. Minimum tokens, maximum output.

## The Hard Parts

There are three problems that make video search genuinely difficult, and they compound each other.

**First, video is multimodal.** There's the visual track (what's on screen), the audio track (what's being said), and the structural layer (scene changes, pacing, chapter boundaries). Any one of these is noisy on its own. Putting them together in a way that produces reliable search results is a coordination problem before it's a machine learning problem.

**Second, transcripts are the easy part — and they're not enough.** Getting a clean transcript out of video is straightforward now. My pipeline gives them to you on demand. But a transcript is just words. "How to make a roux" and "the history of French sauces" share vocabulary but answer different questions. You need to understand what each segment is *about*, not just what words it contains. (Stick around — I'll show you how to pull beautiful, structured transcripts with more than just what's said.)

**Third, speed and accuracy pull in opposite directions.** The brute-force approach — transcribe everything, embed everything, run a large model over every possible segment — works fine. It's also slow and expensive. Getting the same quality on a flash lite model, in under five seconds, requires being clever about what you compute and when.

## The Constraints I Set

I gave myself specific constraints because constraints force design decisions:

| Constraint | Why It Matters |
|---|---|
| Flash lite model | Fastest and cheapest Google has. Could've used GPT Nano, but one provider keeps things simple. Gemini's the most accessible anyway — $300 signup bonus for 90 days |
| <5 second response | Search that takes longer than a page load might as well not exist |
| >95% accuracy | Precision without recall is useless; recall without precision is noise |
| Solo developer | Nobody to delegate to. Every decision had to earn its complexity |
| AI-assisted toolchain | Built with the help of the legendary GPT-4.5. Big Bro — my IP — not only wrote itself, it writes most of my code now. AI as multiplier, not crutch |

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

The target was 6-minute videos — trailers, corporate presentations, cooking shows — the kind of content you find on YouTube. But I stress-tested the pipeline on longer material too. The longest single video I ran through end to end was 38 minutes 😉. It held up.

## Who This Is For

If you're an engineer building search, working with video, or just curious about what's possible when you combine modern NLP tooling with some creative architecture — this is for you. If you're an investor or executive trying to understand why "AI video search" isn't just another buzzword, this is also for you.

This is about architecture, technique, and lessons learned. You should be able to read this series and walk away with enough to build something similar.

Next up: the ingestion pipeline. How do you take hours of raw video and turn it into something a search engine can actually use?
