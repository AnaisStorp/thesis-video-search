# AVA: agentic video analytics with video–language models

**Yan, Y., Jiang, S., Cao, T., Yang, Y., Yang, Q., Shu, Y., Yang, Y., Qiu, L. (2026). AVA: Towards Agentic Video Analytics with Vision Language Models. NSDI 2026. arXiv:2505.00254.** Microsoft Research and partners. Code: `github.com/I-ESC/Project-Ava`. Benchmark: `huggingface.co/datasets/iesc/Ava-100`.

> The thesis subject calls it "AVAS"; the arXiv title says **AVA** (written "Ava" in the HTML version). It's the same paper.

## In one sentence

Running a big video model over a 10-hour video at question time is far too slow. So AVA has a small model describe the video as it streams in, organises those descriptions into a graph of events, and lets an LLM agent search that graph when a question arrives.

## The problem

- Video–language models (VLMs) can answer detailed questions, but they only fit a limited number of frames. On a video of 10+ hours they either sample very sparsely (and miss things) or blow up in cost.
- Plain "video RAG" (cut the video into chunks, caption them, embed the captions, retrieve the top-k) loses how events relate in time and which people or objects appear in which events.

## How it works

### 1. Building the index while the video arrives (near real time)

- Cut the stream into **fixed 3-second chunks**.
- A **small VLM (Qwen2.5-VL-7B)** writes a description of each chunk.
- **Merge neighbouring chunks that say the same thing** into "semantic chunks", using BERTScore similarity between descriptions (threshold about 0.65). So 10 minutes of an empty corridor becomes one event, not 200.
- For each event, the small VLM pulls out **entities** (people, objects) and **relations**, and writes a summary.
- Everything goes into an **Event Knowledge Graph (EKG)**:
  - **Events** in time order, linked by before/after.
  - **Entities** in each event. Duplicates are merged by embedding them (JinaCLIP) and clustering, so "the man in the grey hoodie" in event 3 and event 40 can become one node **(details: to check)**.
  - Links between entities, and between entities and events.
  - Raw frames are also embedded (JinaCLIP) and linked to their events.
- Throughput: the paper says **more than 5 FPS on an edge server with 2× RTX 4090**, so it keeps up with a live stream that's sampled at a few FPS.

### 2. Answering a question (agentic retrieval and generation)

- **Tri-view retrieval**: search the query against three things at once (event descriptions, entities, raw-frame embeddings), then merge the rankings.
- **Agentic search**: an LLM agent looks at what came back and decides what to do next. It can move forward or backward in time from a hit, rewrite the query, or stop and answer. It's a small tree search over the graph.
- **Consistency check**: generate several candidate answers, keep the most consistent one, and if top answers disagree, go back to the **raw frames** with a stronger VLM to settle it.

## Results (from the abstract)

- **LVBench**: 62.3% accuracy. **VideoMME-Long**: 64.1%. Both are better than plain VLMs (GPT-4o, Gemini-1.5-Pro, Qwen2.5-VL and others) and earlier video-RAG systems (VideoAgent, VideoTree, DrVideo and others).
- They built **AVA-100**: **8 videos, each longer than 10 hours, with 120 hand-written question–answer pairs**. The videos cover daily activities, city walks, **traffic monitoring** and wildlife monitoring. AVA gets **75.8%**.
- Query-time cost is real: agentic search plus answer generation takes on the order of **minutes per question** with open models on an A100 **(exact latency table: to check)**.

## Why it matters for my thesis

This is the **blueprint for system 3**:

- **Ingest time**: a small VLM describes footage → events → index. That's an ongoing cost per hour of footage per camera. It is *the* number to measure for the "as the archive grows" question.
- **Query time**: cheap retrieval gives a short list → an expensive VLM re-ranks or checks only those candidates.

### What's different in my setting (this is my contribution)

| AVA | My thesis |
|-----|-----------|
| A few long videos, often moving cameras | **Many fixed cameras** at once |
| Question answering (multiple choice / free text) | **Retrieval**: return the right clips, measured with recall@k |
| No structured filters | **Hard filters on camera, zone and time** ("gate 3 after 6 pm") |
| Accuracy is the main metric | **Accuracy and cost and latency**, at several archive sizes |
| Web / benchmark videos | Real fixed-camera footage from Whiteshield |

Ideas to borrow directly:

- **Merging similar chunks** is perfect for fixed cameras. Most CCTV footage is "nothing happens", so this could cut ingest cost a lot. I could also gate the VLM with motion or the detector: only describe chunks where something moves.
- **Entity merging** is close to cross-camera re-ID, which is **out of scope** (no live cross-camera tracking). Keep it within one camera, or turn it off, and explain why in the ethics section.

### Weak points to measure

- Small VLM descriptions miss small or distant things and get colours wrong in bad light.
- If something wasn't described at ingest, it can never be found later, unless the raw-frame embeddings save you. That's why the tri-view design matters.
