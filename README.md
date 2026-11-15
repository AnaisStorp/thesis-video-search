# Finding footage by description

**Master's thesis: attribute indexes, image–text embeddings and video–language models for natural-language video search over fixed-camera archives.**

> **Research question:** Which approach, or which combination, answers natural-language queries over fixed-camera archives accurately enough, at acceptable cost and speed, as the archive grows from days to months of footage?

Reading notes for all the papers are in [`docs/`](docs/README.md).

---

## 1. The idea in one picture

```
                         ┌─────────────────────── INGEST (as footage arrives) ───────────────────────┐
  cameras ──► video ──►  │ detector + tracker ──► S1: attribute classifier ──► SQL table (tracks)    │
                         │        │                                                                  │
                         │        └──► crops / frames / clips ──► S2: PE encoder ──► Faiss index     │
                         │                                                                           │
                         │ motion-gated chunks ──► S3: small VLM captions ──► event index (+ embeds) │
                         └───────────────────────────────────────────────────────────────────────────┘
                         ┌──────────────────────────── QUERY ───────────────────────────────────────┐
  "red jacket near ──►   │ LLM parser ──► {camera/zone, time range, class, attributes, free text}    │
   gate 3 after 6pm"     │      └──► ROUTER ──► S1 (filter) │ S2 (vector search) │ S3 (re-rank top-N)│
                         │                                     ──► ranked clips + audit log          │
                         └───────────────────────────────────────────────────────────────────────────┘
```

All three systems share the same **front end** (query parsing into hard filters + free text) and the same **output unit** (a clip: `camera, t_start, t_end`, optionally a bounding box). That makes them directly comparable and lets them be combined.

---

## 2. Plan of work

Rough timeline for about 6 months. Adjust to the real internship dates.

### Phase 0: Setup and framing (weeks 1–3)
- [ ] Read the 4 core papers ([docs](docs/README.md)) and write the related-work chapter skeleton.
- [ ] Meet operators: **collect real queries** they've asked after past incidents (anonymised). This is the most valuable input to the whole thesis.
- [ ] Legal: DPIA / data access agreement, retention rules, on-prem only (see [docs/06](docs/06_privacy_and_eu_ai_act.md)).
- [ ] Get access to footage, camera metadata and GPU. Set up the repo, environment and storage layout.
- [ ] Build a small **dev set on public data** (VIRAT / MEVA / UCF-Crime + UCA) so I'm not blocked while waiting for company data.

### Phase 1: Test set (objective 1, weeks 3–8)
- [ ] Choose a **core subset**: e.g. 10–20 cameras × 3–7 days, with varied scenes (gate, parking, loading bay, indoor corridor) and lighting (day, night, IR).
- [ ] Write **200–400 queries** from:
  - real operator queries (paraphrased);
  - queries I write from footage I've watched;
  - targeted queries for each failure type.
- [ ] Tag each query with **categories** (multi-label):
  `attribute-colour`, `attribute-object` (bag, helmet…), `vehicle-type`, `action`, `count`, `spatial-relation`, `place/zone`, `time`, `rare-word`, `negation`, `multi-entity`.
- [ ] **Paraphrase sets**: 3–5 phrasings per query (some written by colleagues, some by an LLM in different styles, following the idea in [docs/02](docs/02_ham_text_to_person_reid.md)) to test robustness to wording.
- [ ] **Relevance labels**, done by **pooling** (as in TREC):
  1. run all early versions of the three systems + time and camera filters;
  2. collect the union of their top-k;
  3. humans judge each candidate as relevant, partly relevant or not;
  4. add known positives found by hand (from incident reports or my own watching) to catch what all systems missed.
- [ ] Ground truth unit = time interval on a camera (+ optional box). A hit = a returned clip overlapping a relevant interval by at least X% (define X, e.g. temporal IoU ≥ 0.3 or "contains the moment").
- [ ] Annotation guide + **inter-annotator agreement** on a sample (Cohen's κ).
- [ ] Split: a **dev** set for tuning (~30%) and a **test** set kept untouched (~70%).

### Phase 2: Build the three systems (objective 2, weeks 6–14)

Shared pipeline:
- [ ] Video decoding at a fixed sampling rate (e.g. 1–2 fps for indexing).
- [ ] Detector + tracker (reuse the company's detector if possible, otherwise YOLO + ByteTrack) → tracks with `camera, track_id, class, t_start, t_end, boxes`.
- [ ] Query parser: LLM → JSON `{cameras/zones, time range, object class, attributes, free_text}`. Check it separately (parse accuracy).
- [ ] Face blurring on stored crops.

**S1: Structured attribute index**
- [ ] Fix the attribute schema (from PA-100K / RAP and vehicle colour/type, minus sensitive traits).
- [ ] Per-track attributes: either a PAR classifier on the best crops, or a VLM filling a fixed JSON form (compare both if time allows).
- [ ] Store in Postgres/SQLite. Query = SQL filter + ranking by attribute confidence.

**S2: Image–text embedding index**
- [ ] Backbone: **PE Core** (B and L, maybe G) + one alternative (SigLIP 2) for comparison.
- [ ] Three granularities: **track crops** (best K crops per track), **whole frames**, **short clips** (frame-averaged).
- [ ] Faiss: `Flat` (ground truth) → `IVF`, `HNSW`, `IVF-PQ`. Time and camera filters via per-shard indexes or `IDSelector`.
- [ ] Score fusion when several granularities are used.

**S3: Video–language event index (AVA-style)**
- [ ] Ingest: chunks of a few seconds → **motion- or detector-gated** (skip empty chunks) → small VLM (e.g. Qwen2.5-VL-7B or newer) writes a description → merge similar consecutive chunks into events → store text + text embedding + frame embedding.
- [ ] Query: retrieve top-N events (text and frame embeddings + filters) → **re-rank / verify** the top-N with a VLM looking at the actual frames.
- [ ] Optional: agentic search (move forward and back in time, rewrite the query) as in AVA. Measure whether it's worth its cost.

### Phase 3: Evaluation at several archive sizes (objective 3, weeks 12–18)
- [ ] **Archive sizes**: 1 day → 1 week → 1 month → (3 months if retention and storage allow). The relevant clips stay the same, and I add **distractor footage** to grow the archive. This isolates the effect of scale.
- [ ] **Metrics**:
  - Accuracy: Recall@1/5/10/50, mAP, and recall per query category.
  - Speed: query latency p50 / p95, split into parse, retrieval and re-rank.
  - Ingest cost: **GPU-seconds per hour of footage** per camera, throughput in real-time factor.
  - Query cost: GPU-seconds (or €) per query.
  - Storage: bytes per hour of footage (DB, vectors, text).
  - Faiss: ANN recall vs exact, memory, build time.
- [ ] Convert GPU time to € with one fixed price per GPU-hour (state the assumption).
- [ ] Plot **accuracy vs cost** and **accuracy vs archive size** curves for each system and combination.
- [ ] Combinations to test:
  - S1 filter → S2 ranking;
  - S2 short list → S3 re-rank;
  - S1 ∪ S2 → S3.

### Phase 4: Failure analysis and router (objective 4, weeks 16–21)
- [ ] Break down recall by query category × system → a table showing **which system fails on what** (colour, counting, positions, rare words, night/IR, small or distant objects).
- [ ] Look at failures qualitatively (with example frames, faces blurred).
- [ ] **Router**: given a parsed query, choose the cheapest system likely to answer it. Options, from simple to fancy:
  1. **Rule-based**: e.g. "only schema attributes + filters → S1; free-text appearance → S2; actions, counts, relations → S2 + S3 re-rank".
  2. **Learned**: a small classifier on query features (categories, parse output, S2 score distribution) trained on dev-set outcomes.
  3. **Cascade**: try the cheap system first, and move up if confidence is low (as in FrugalGPT / Focus).
- [ ] Evaluate the router on the test set: accuracy vs average cost, compared with "always S1/S2/S3" and an **oracle** router (upper bound).

### Phase 5: Writing (weeks 20–26)
- [ ] Chapters: Introduction → Related work → Setting and test set → Systems → Results → Failure analysis and router → Ethics and privacy → Conclusion.
- [ ] Optional extension: **Arabic queries**. Translate a subset of the test set and check each system (PE text encoder, multilingual LLM parser, VLM). The cheap baseline is to translate the query to English first.

---

## 3. Data I need from the company

### 3.1 Video
| Item | Detail to ask | Why |
|------|---------------|-----|
| Raw footage | Number of cameras, **total hours available**, how far back | Defines max archive size |
| Format | Codec, resolution, fps, bitrate, how it's stored (NVR/VMS export?) | Decoding pipeline, cost estimates |
| Camera variety | Indoor/outdoor, wide/narrow view, height, **night/IR**, PTZ (moving cameras are out of scope) | Test-set diversity, colour queries at night |
| Continuity | Continuous recording or motion-triggered? Gaps? | Affects the ingest cost model |

### 3.2 Metadata
- **Camera list**: ID, site, human-readable name ("Gate 3"), indoor/outdoor.
- **Zones / map**: which camera sees which named place (gate, loading bay, car park), ideally polygons per camera image. *Needed for "near gate 3".*
- **Accurate timestamps** + time zone + clock drift info. *Needed for "after 6 pm" and "yesterday".*
- **Existing detector logs** (person / car events with timestamps), which are the current baseline the thesis must beat.

### 3.3 Queries and ground truth (most important!)
- **Real operator queries** from past investigations: the wording as typed or said, plus the answer clip if known. Even 30–50 real ones give the test set credibility.
- **Incident reports** that point to a time and camera (used as known positives).
- **Annotator time**: a few hours from 2–3 colleagues for relevance judgments and agreement checks.
- Operator **vocabulary**: site-specific words ("the blue door", "dock 2", company vehicle names). These are the "rare words".

### 3.4 Compute and infrastructure
- GPU(s) on-prem: type, count, memory (e.g. 1–2 × A100/H100 or 4090s), and how much **storage** (raw video for months is TBs).
- Whether calling external APIs is allowed (probably **no** for footage, which means open-source VLMs only).
- An approved **€/GPU-hour** figure (or cloud equivalent) for the cost comparison.

### 3.5 Legal and organisational
- Data access agreement / DPIA, **retention period** (limits how long the index and test set can live), who can see raw footage, rules for figures in the thesis (blurred, no faces).
- Confidentiality: what can be published (can I release the query set without video? only aggregate numbers?).

### 3.6 If company data is late or limited
Develop on **VIRAT, MEVA, UCF-Crime + UCA, CityFlow-NL** (see [docs/05](docs/05_other_relevant_work.md#d-datasets-with-fixed-cameras-to-develop-before-company-data-arrives-and-for-public-reproducibility)). A public mini-benchmark built on MEVA (multi-camera, long) would also make the thesis reproducible, which is a nice bonus contribution.

---

## 4. Expected contributions

1. A **fixed-camera natural-language search benchmark**: queries tagged by category, paraphrase sets, pooled relevance labels, at several archive sizes.
2. A **like-for-like comparison** of attribute index vs embedding index vs VLM event index on **accuracy, latency, ingest cost and query cost**, as the archive grows.
3. A **failure taxonomy**: which query types break which approach.
4. A **cost-aware router** that sends each query to the cheapest approach likely to answer it, measured against an oracle.
5. A **privacy-by-design** argument: appearance and context search without biometric identification, with audit logging.

---

## 5. Proposed repo layout

```
thesis-video-search/
├── README.md              ← this plan
├── docs/                  ← reading notes
├── data/                  ← (git-ignored) footage pointers, metadata, never raw video in git
├── benchmark/             ← queries, categories, paraphrases, relevance labels, annotation guide
├── src/
│   ├── common/            ← decoding, detection, tracking, query parser, clip format, audit log
│   ├── s1_attributes/
│   ├── s2_embeddings/
│   ├── s3_vlm_events/
│   ├── router/
│   └── eval/              ← metrics, cost accounting, archive-size scaling
├── experiments/           ← configs + result logs
└── thesis/                ← LaTeX manuscript
```

## 6. Open questions to settle early
- What is the **unit of an answer**: a track, a fixed-length clip, or an event? (It affects every metric.)
- What **archive length** can legally and physically be kept? (Retention vs "months of footage".)
- Will the company's **existing detector** be used, or my own?
- How much **operator time** can I get for writing queries and judging relevance?
