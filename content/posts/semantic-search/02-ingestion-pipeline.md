+++
title = '2. The Ingestion Pipeline: Turning Hours of Video Into Structured Metadata'
date = '2026-05-31'
draft = false
tags = ['semantic-search', 'pipeline', 'nlp']
categories = ["NLP", "Video Search"]
series = ["Semantic Video Search"]
+++

Search is only as good as what you feed it.

You can have the cleverest embedding strategy in the world. The fastest vector store. The most elegant query engine. None of it matters if your ingestion pipeline produces garbage.

In the last post, I talked about *why* I built a semantic video search engine. This post is about the first and most important step: turning raw video into something a machine can actually search.

That means frames and structured metadata — extracted fast, extracted accurately, and extracted without the video overstaying its welcome. Remember: upload, extract, purge. The ingestion pipeline is where that promise lives or dies.

There's a business reason for getting this right too. If your pipeline leaks — if it keeps video longer than it should, if it phones home to a third-party API you didn't audit, if it burns compute on frames nobody will ever search — you're not just wasting money. You're carrying risk. For clients who mandate their content never exists off-prem, a sloppy pipeline is a dealbreaker.

So this post is about the engineering, but it's also about the architecture decisions that make the whole system trustworthy. I'll walk through four stages:

1. **Frame Extraction** — which frames matter, and which ones you can safely ignore
2. **Scene Analysis** — extracting entities, actions, camera, mood, and environment directly from frames
3. **Structuring** — scene boundaries, segmentation, and assembling the metadata
4. **Embedding** — turning structured concepts into vectors the search engine can actually use

If you're an engineer, the details are right below. If you're evaluating whether a system like this can handle your data without creating liability, this is the post that answers that.

## The Pipeline at a Glance

```
Raw Video → Frame Extraction → Scene Analysis → Structuring → Embedding → Vector Store
```

## What Makes Semantic Search Actually Work

Before I walk through each stage, it helps to know what the pipeline needs to produce. If the ingestion pipeline is a factory, we need to define the product — not as a schema, but as a set of capabilities.

Here's a real query: *"Find me 5 Keanu Reeves fighting scenes."*

That's it. That's the whole prompt. No filters, no facets, no engineered query syntax. Just natural language. And the system has to return five distinct, relevant clips from different movies — not five timestamps from John Wick Chapter 4.

Here's another: *"Find me scenes with an orange car."*

And another, from a security context: *"Find me scenes where the robber blows the door open."*

To answer queries like these, the ingestor has to extract five categories of information from every video:

**Entities — who and what is in the frame.** People (Keanu Reeves, a masked man), objects (orange Ford Mustang, a crowbar), colours, clothing, identifying features. This is what turns "orange car" into a searchable concept instead of a guess.

**Actions — what's happening.** Fighting, racing, drifting, a high-speed chase, blowing a door open. More than just motion detection. The difference between "person running" and "person fleeing" is intent, and intent is what the query cares about.

**Camera — how the footage was captured.** CCTV, hidden camera, high angle, drone shot, handheld. This matters more than people think. A query like "riot footage from street-level cameras" fails completely if the ingestor didn't tag the camera type.

**Mood and Environment — the atmosphere of the scene.** Intense, mysterious, calm. Mountains, train tracks, urban alleyways. These are the hardest to extract because they're subjective, but they're also what makes search feel magical when it works.

**Structure — how the video is organised.** Scene boundaries, topic shifts, segment durations. Where does one subject end and another begin? This is what lets you return *segments* instead of timestamps. Nobody wants to scrub a timeline.

One thing the ingestor does *not* produce: a transcript.

Transcripts seem useful. They're the obvious output of any video pipeline. But for semantic search, they're actively harmful. A transcript conflates "Keanu Reeves explaining fight choreography" with "Keanu Reeves performing fight choreography" — same vocabulary, completely different meaning. Embedding a transcript gives you keyword search with extra steps. Embedding what the scene is *about* gives you actual semantic understanding.

So the ingestor produces metadata directly from the frames. No intermediate transcript. No text passthrough. Raw video in, structured concepts out.

## The Magic: Letting the AI Watch the Video

If you're thinking "extract one frame every N seconds," you're already wrong. That approach gives you a stitched-up mess — choppy scenes, missed transitions, and a search experience that feels like scanning a flipbook with half the pages torn out.

I know because I tested it — and I was left genuinely aghast at how bad it is. Hard time-bound scene detection produces results that are technically correct and completely useless. A scene that starts at 4.7 seconds lands on a frame at 5.0 seconds. That 0.3-second gap contains the character reveal, the explosion, the moment the audience gasps. Congratulations — your search engine just indexed everything except the thing someone wanted to find.

The right approach was clear: let the AI decide.

### How It Works

The system sends each video to a multimodal model with three things:

1. **The video itself** — every frame, at whatever resolution the model can consume
2. **The filename** — sounds trivial, but "John Wick Chapter 4.mp4" tells the model which universe it's in, which characters to expect, which tropes to look for. If the filename is meaningless (like "clip_0042.mp4"), the model is instructed to ignore it and work purely from the visuals.
3. **The total duration** — a hard boundary so the model knows the playing field. No scene can exceed 30 seconds. No scene can extend past the video's end. Scene one always starts at zero.

The model watches the entire video — however Gemini does it, that's their magic, not mine — and returns a structured breakdown: scene start times, end times, and everything each scene contains. Entities. Actions. Camera angles. Mood. The full five-category metadata stack.

This is the most critical decision in the whole pipeline. If this step gets the scene boundaries wrong, every downstream system suffers. You can have the cleverest embedding strategy in the world — if the frame extractor chopped a fight scene in half, the search engine will return two half-relevant results instead of one perfect one. Garbage in, garbage out.

### The Speed Problem

AI-driven frame extraction is accurate. It is also slow when you have hours of video to process, and the rest of the pipeline is waiting on its output.

The solution was concurrency at the model level. Multiple goroutines handle extraction jobs independently — single responsibility principle: one goroutine, one video. A round-robin across available models prevents any single model from hitting its tokens-per-minute limit. When a model does get rate-limited, the system backs off gradually and retries, coordinated through Redis. No deadlocks. No dropped jobs. No manual intervention.

The tradeoff is upfront: you spend more compute at the ingestion stage than a naive frame-sampling approach would. But you get it back tenfold at query time, because the search engine has clean scene boundaries to work with instead of arbitrary timestamps.

### What the Prompt Actually Does

A good prompt is an engineering artifact. It specifies not just what to extract, but how to structure it so every downstream stage speaks the same language. Ambiguity at this stage compounds into wrong search results at the other end.

The prompt instructs the model to identify entities with dual-layer specificity ("Tyrannosaurus Rex (Dinosaur)"), tag camera work (close-ups, wide shots, fourth-wall breaks), recognise cinematic tropes, describe mood and atmosphere, and explicitly note the audio state. It knows the difference between "no dialogue" and "music only." It knows a car isn't just a car — it's an "orange Ford Mustang (Car/Vehicle)."

But the principle is what matters: the AI decides what matters, when scenes change, and how to label them. The alternative — writing rules for every edge case — doesn't scale across different video types, genres, and use cases. The AI generalises. That's its job. The job was building the system around it so it generalises fast, accurately, and without melting down under load.

This pipeline feeds every stage that follows. Structuring depends on clean scene boundaries. Embedding depends on rich, accurate metadata. The search engine depends on all of it. Get this right, and everything downstream has a fighting chance. Get it wrong, and the cleverest query engine in the world can't save you.

