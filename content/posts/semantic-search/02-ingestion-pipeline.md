+++
title = '2. The Ingestion Pipeline: Turning Hours of Video Into Structured Metadata'
date = '2026-05-31'
draft = true
tags = ['semantic-search', 'pipeline', 'nlp']
series = ['Semantic Video Search']
+++

## The Pipeline at a Glance

```
Raw Video → Frame Extraction → Transcription → NLP Processing → Structured Metadata → Vector Store
```

## Step 1: Frame Extraction
- [Keyframe selection strategy — not every frame, just the right ones]
- [Scene detection: when does the topic change?]
- [Resolution vs. speed tradeoffs]

## Step 2: Transcription
- [Whisper or equivalent: accuracy at speed]
- [Speaker diarisation: knowing who said what]
- [Language detection, punctuation, formatting]

## Step 3: NLP Processing
- [Chunking: how to split long transcripts into searchable segments]
- [Entity extraction: people, topics, concepts]
- [Summarisation: generating descriptions at multiple granularities]

## Step 4: Metadata Structure
- [What fields matter for search? Title, segment, speaker, entities, summary]
- [How do you store it so queries can hit it efficiently?]

## Design Decisions
- [Why this pipeline vs. alternatives]
- [What I would change if I rebuilt it]

---

**Status:** Outline.
