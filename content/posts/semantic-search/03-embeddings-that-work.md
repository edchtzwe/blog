+++
title = '3. Embeddings That Actually Work: Vector Strategies for Video Content'
date = '2026-07-26'
draft = false
tags = ['semantic-search', 'embeddings', 'vector-db']
categories = ["NLP", "Video Search"]
series = ["Semantic Video Search"]
+++

You can't search what you can't represent.

In the last post, I walked through the ingestion pipeline — how raw video becomes structured metadata: entities, actions, camera work, mood, environment. But metadata alone doesn't give you search. A database of JSON blobs is just a database. To make it searchable, you need to turn that metadata into vectors — numerical representations where similar concepts cluster together in space.

This sounds straightforward. It isn't.

The embedding strategy is where most semantic search projects go wrong. Not because embedding models are hard to use — they're trivially easy now. But because most people embed the wrong thing, in the wrong way, and then wonder why their search engine returns garbage.

I went down several dead ends before landing on an approach that delivered >95% accuracy in under 5 seconds. This post is about what I tried, what failed, and what actually worked.

## The Naive Approach (And Why It Fails)

The obvious thing to embed is the transcript. It's text. Text is what embedding models were built for. What could go wrong?

Everything, as it turns out.

A transcript conflates "Keanu Reeves explaining fight choreography" with "Keanu Reeves performing fight choreography." Same vocabulary. Same entities. Completely different semantic meaning. When someone searches "fight scenes in John Wick," they want the fighting, not the behind-the-scenes documentary where he talks about how the fights were staged.

But transcripts have a deeper problem. They're a lossy compression of what's on screen. A 30-second scene might contain three sentences of dialogue and fifteen seconds of silent action. The transcript captures the three sentences. It misses the action entirely — the part someone searching is actually looking for.

I tested transcript-only embeddings early on. The results were usable for keyword-adjacent queries ("who talks about the car") and useless for everything else. The accuracy was embarrassing. I stopped measuring after I saw the same false positive for the tenth time.

## What to Embed Instead

The breakthrough came when I stopped thinking about embedding text and started thinking about embedding *concepts*.

Each scene in my pipeline produces a structured metadata object: entities present, actions occurring, camera angles, mood, environment, timestamps. This is the output of the ingestion stage — a rich, multi-layered description of what the scene contains.

What I embed is a *synthesised representation* of that metadata. Not the raw JSON. Not a transcript. A natural-language summary of what the scene is about, generated from the structured data.

For example, a 45-second scene from a car chase might produce an embedding prompt like:

> A high-speed car chase through urban streets at night. Multiple vehicles: a black Ford Mustang, a silver sedan, a police cruiser. Actions: drifting, near-collisions, a crash into a storefront. Camera: handheld, frequent cuts, close-ups on driver faces. Mood: tense, urgent. Environment: city streets, neon signs, wet pavement.

This is what gets embedded. A scene-level summary that captures the visual and atmospheric content, not just the dialogue.

The result: similarity search now works the way humans think. "Cars drifting at night" returns the right scene even if no one ever said those words on screen. The embedding space maps to *what's happening*, not *what was said*.

## Embedding Model Selection

I started with Gemini 2.5 Flash Lite. Simple reasoning: Google's models already understood video context better than anything else I'd used, so it made sense to stay in the same ecosystem for both ingestion and embedding.

Early experiments were promising. Gemini 2.5 Flash Lite produced embeddings that captured scene-level semantics well — the clusters made intuitive sense. Fight scenes grouped with fight scenes. Dialogue-heavy scenes grouped away from visual spectacle.

The first diversion was Cohere Rerank. I'd read about reranking as a way to improve search relevance — the idea being you do a coarse semantic search, then feed the candidate set through a reranker that applies LLM-grade reasoning to reorder results. In theory, this is semantic search *plus* logic. In practice, Cohere Rerank turned out to be a weaker vector embedder than Gemini for my use case. The clusters were mushier, the top-k results less precise. It wasn't *bad*, just worse. And when you're chasing sub-5-second latency, "worse" doesn't survive.

I ended up where I started, but expanded: Gemini 2.5 Flash Lite, Gemini 2.5 Flash, Gemini 3 Flash, and Gemini 3 Flash Lite. Four models from the same family, round-robined across requests to stay under rate limits while maximising throughput. No exotic embeddings. No custom fine-tuning. Just solid model selection and enough TPM headroom to never stall.

The key insight: model selection matters less than *what* you embed. A mediocre model embedding the right representation outperforms a great model embedding the wrong thing.

## Vector Store Architecture

Vectors need somewhere to live. I looked at three options:

- **PostgreSQL with pgvector** — dead simple if you're already on Postgres, and we were. The pgvector extension has matured significantly: IVFFlat indexing for approximate nearest neighbour, reasonable performance at moderate scale, zero operational overhead.
- **Pinecone** — studied it. Fully managed, slick API, but the cost curve bends the wrong way at production volume. Good for demos, expensive for deployment.
- **Qdrant** — seriously considered. Self-hosted, HNSW indexing, stronger at hybrid search. But introducing a new infrastructure component for one feature didn't pass the tradeoff test.

I went with PostgreSQL and pgvector. The operational simplicity was the deciding factor — we already run Postgres, we already back it up, we already monitor it. Adding a vector index was a migration, not a new deployment. No new service to maintain, no new failure mode to debug at 2 a.m.

At our scale — hundreds of thousands of vectors per video library — pgvector with IVFFlat indexing handles approximate nearest neighbour queries comfortably within the latency budget. The difference between pgvector and a dedicated vector database only becomes meaningful at an order of magnitude more data than we're dealing with. Choose simplicity first. Optimise when you have proof you need to.

## The Embedding Content String

Here's a decision that saved me from a common trap: don't store JSON.

Every embedding has metadata attached to it — video name, actors, genre, camera style, mood, environment. The obvious thing is to store it as structured JSON, index the fields, and build rich filter queries. That works. It also dumps a pile of tokens into every LLM call downstream, and most of them are noise.

Instead, I embed a pure string — a constructed representation that concatenates the relevant metadata fields into one dense text blob. Something like:

> Video: John Wick: Chapter 4. Scene: Club shootout, 01:23:45–01:27:12. Cast: Keanu Reeves, Donnie Yen. Genre: Action. Camera: Steadicam, wide angles. Mood: Tense, energetic. Environment: Nightclub, neon, crowds.

Every field that matters for search goes into the string. No nesting. No brackets. No JSON. Just a flat, token-efficient representation that the embedding model can digest directly.

The search flow is then:

1. **Semantic search** — the user's query is embedded, pgvector returns the top-k closest scene vectors.
2. **LLM filter** — the candidate set goes to the LLM with the original prompt. The model checks each result for *real* relevance, not just vector proximity. False positives get eliminated here.

This two-stage approach — semantic search for recall, LLM for precision — gives you the best of both worlds. The semantic search casts a wide net. The LLM throws back what doesn't belong. And because the embedding strings are dense and token-lean, the LLM pass is fast.

Storage is cheap. Latency and user retention are what matter. There's no need for multi-level indexing — just pack enough metadata into each embedding string and you achieve the same result at the query level. If someone searches "Keanu Reeves fight scenes," the embedding string already contains actor names and scene descriptions. The semantic search naturally narrows the field. No coarse-to-fine gating required.

## The Results

With this embedding strategy:

- **Recall:** >95% on test queries — the right scene appears in top 5 results
- **Latency:** <5s end-to-end, including the LLM filtering pass
- **Storage:** ~2MB per hour of video (embeddings + metadata)

More importantly: the search feels right. Queries that should work, work. The edge cases that broke transcript-only search — ambiguous phrasing, visual-only scenes, multi-concept queries — now return relevant results.

Next post, I'll dig into the model itself: how to run production-grade semantic search on a flash lite model without sacrificing accuracy. Spoiler: it's not about the model size. It's about how you use it.
