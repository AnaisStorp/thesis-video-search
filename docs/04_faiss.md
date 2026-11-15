# The Faiss Library

**Douze, M., Guzhva, A., Deng, C., Johnson, J., Szilvasy, G., Mazaré, P.-E., Lomeli, M., Hosseini, L., Jégou, H. (2024, revised 2025). The Faiss Library. arXiv:2401.08281.** Meta FAIR. Library: `facebookresearch/faiss`.

## In one sentence

This is the "manual plus design rationale" for Faiss, the vector-search library most vector databases build on. Its main message: **you always trade off speed, memory and accuracy, and here are the knobs.**

## The basic problem

I have N vectors (e.g. one per person crop) and a query vector. I want the k closest ones.

- **Exact (brute force)**: compare with all N. It's perfect, but the time grows linearly with the archive. It's fine for a few million vectors on a GPU, and too slow or memory-heavy beyond that.
- **Approximate nearest neighbour (ANN)**: accept missing a few true neighbours in exchange for being 10–1000× faster or smaller.

## The main families of index (the ones I'll actually use)

| Index | Idea in plain words | Good for | Cost |
|-------|---------------------|----------|------|
| `Flat` | Compare against everything | Ground truth, small archives | Memory: 4·d bytes per vector. Time grows with N. |
| `IVF` | Run k-means to split space into `nlist` cells, and only search the `nprobe` closest cells | Large archives | Accuracy depends on `nprobe`. Needs training. |
| `PQ` (product quantization) | Cut each vector into m pieces and replace each piece with a 1-byte code | Huge archives in RAM | Vectors shrink from KBs to tens of bytes. You lose some accuracy. |
| `IVF-PQ` | The two above combined | The classic "billion vectors" setup | Tune `nlist`, `nprobe`, `m` |
| `HNSW` | A graph where you hop from neighbour to neighbour | Very fast and very accurate | Uses more memory (graph links). Slow to build. Hard to delete. |
| Scalar quantization (`SQ8`, `SQfp16`) | Store floats as 8 or 16 bits | Easy 2–4× memory saving | Very small accuracy loss |

The paper also covers GPU indexes, re-ranking (search with a compressed index, then re-score the top candidates with exact vectors) and how to benchmark properly.

## How to compare indexes (the paper's method, which I should copy)

- Plot **accuracy (recall@k against the exact neighbours) vs search time** for many parameter settings, and keep the **Pareto front**.
- Report **memory per vector** and **build/training time** separately.
- The `index_factory` strings (e.g. `"IVF4096,PQ32"`) make it easy to sweep many configurations.

## Why it matters for my thesis

Back-of-the-envelope numbers to make the "archive grows" question concrete (to redo with real numbers):

- Suppose 1 embedding per second per camera (frame-level) and d = 1024 floats → 4 KB per vector.
- 1 camera × 1 day = 86,400 vectors ≈ **350 MB** as Flat float32.
- 50 cameras × 90 days ≈ 390M vectors ≈ **1.6 TB** as Flat. That doesn't fit in RAM.
  - With PQ at 64 bytes per vector it's about 25 GB. That fits.
- With per-object crops instead of frames, the count depends on traffic: an empty corridor gives zero, a busy gate gives a lot.

So **the index choice changes with archive size**. That's exactly the scaling experiment in objective 3: Flat at 1 day, IVF/HNSW at 1 week, IVF-PQ at 1–3 months.

### Two important practical points

1. **Filtering by time and camera.** Operators always say "gate 3 after 6 pm". Options:
   - Pre-filter: Faiss supports restricting search to a subset of IDs (`IDSelector`).
   - One index per camera per day: simple and naturally sharded.
   - Post-filter: search more, then drop what doesn't match. It's risky, because with a tight filter you may end up with nothing.

   Which is fastest depends on how selective the filter is. Worth measuring.
2. **Two different "recalls".** ANN recall (did Faiss find the true nearest vectors?) is not the same as task recall (did I find the right clip?). Keep them separate in the thesis. Usually the embedding model, not Faiss, is the bottleneck.

## Related

- Jégou, Douze, Schmid (2011). *Product quantization for nearest neighbor search*. TPAMI. The original PQ paper.
- Malkov & Yashunin (2018). *HNSW*. TPAMI.
- Subramanya et al. (2019). *DiskANN*. NeurIPS. A graph index stored on SSD, relevant if the archive no longer fits in RAM.
- Vector databases built on these ideas (Milvus, Qdrant, pgvector, etc.) handle metadata filters more conveniently. That's a possible implementation choice, but Faiss gives cleaner measurements.
