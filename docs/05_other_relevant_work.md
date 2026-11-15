# Other relevant work

These are shorter notes than the four main papers, grouped by which part of the thesis they help with. Each entry says what it is and why I care. **References are from memory, so verify authors, years and venues before citing.**

---

## A. For system 1: attribute index over tracked objects

### Detection and tracking (the base for systems 1 and 2)
- **YOLO family (Ultralytics YOLOv8 / YOLO11)**: a fast object detector. It is probably close to what is already deployed in practice.
- **ByteTrack** (Zhang et al., ECCV 2022) and **BoT-SORT** (Aharon et al., 2022): simple multi-object trackers that link detections into tracks within one camera. A *track* is the unit I index ("one person walking through camera 4 from 18:02 to 18:03").
- **Grounding DINO** (Liu et al., ECCV 2024) and **OWL-ViT / OWLv2** (Minderer et al., 2022/2023): open-vocabulary detectors. You give a text ("white van") and they find boxes. They sit between systems 1 and 2 and could serve as a baseline or a re-ranker.

### Pedestrian attribute recognition (PAR)
- Datasets: **PETA** (Deng et al., 2014), **RAP / RAPv2** (Li et al., 2016/2019), **PA-100K** (Liu et al., 2017). These are crops of pedestrians labelled with tens of binary attributes: gender, age group, upper/lower clothing type and colour, bag, backpack, hat, etc. **They define the fixed list for system 1.**
- Models: standard CNN or ViT multi-label classifiers. More recently, CLIP-based PAR (e.g. *PromptPAR*, Wang et al., 2023/2024) uses text prompts per attribute.
- Survey: Wang, X. et al. *Pedestrian Attribute Recognition: A Survey* (Pattern Recognition, 2022).

### Vehicle attributes
- **VeRi-776** (Liu et al., 2016) and **VehicleID**: vehicle re-ID with colour and type labels.
- Colour and type classification is a well-solved problem on good crops, and much harder at night (IR cameras turn everything grey, so "red car" can't be answered at all).

**Takeaway for me:** system 1 = detector → tracker → per-track attribute classifier (or a VLM filling a fixed form) → a SQL table with `camera, zone, t_start, t_end, class, attributes…`. A query is parsed by an LLM into a SQL filter.

---

## B. For system 2: text–image / text–video retrieval

### Text-based person search (more than note 02)
- **IRRA** (Jiang & Ye, CVPR 2023): *Cross-Modal Implicit Relation Reasoning and Aligning*. A strong CLIP-based baseline for CUHK-PEDES.
- **RDE** (Qin et al., CVPR 2024): handles noisy text–image pairs.
- **PLIP** (Zuo et al., 2023) and **UniPT / APTM** (Yang et al., ACM MM 2023): pre-training on large synthetic person–text data, the same direction as note 02.

### Vehicle search by language (very close to my setting!)
- **CityFlow-NL** (Feng et al., 2021) and the **AI City Challenge "natural-language vehicle retrieval" track** (2021–2022): **fixed traffic cameras**, with tracked vehicles described in natural language ("a white sedan turns left at the intersection"). It's the best public analogue for vehicle queries like "white van at the loading bay".

### Text-to-video retrieval
- **MSR-VTT** (Xu et al., 2016), **ActivityNet Captions**, **DiDeMo**: short web clips with captions. These are the benchmarks the subject says don't match my setting.
- **CLIP4Clip** (Luo et al., 2022): the simple recipe of embedding frames with CLIP and averaging them. It's a good baseline for clip-level embeddings.
- **InternVideo2** (Wang et al., ECCV 2024) and **VideoPrism** (Zhao et al., ICML 2024): large video foundation encoders. They're alternatives to PE for clip embeddings.
- **SigLIP / SigLIP 2** (Zhai et al., ICCV 2023; Tschannen et al., 2025): a CLIP variant with a sigmoid loss. It's a strong alternative backbone and could be my "second embedding model" for comparison with PE.

### Known weaknesses of CLIP-style models (inputs to objective 4)
- **ARO**: Yuksekgonul et al., *When and why vision-language models behave like bags-of-words* (ICLR 2023). They are weak on word order, relations and attributes bound to the wrong object.
- **Winoground** (Thrush et al., CVPR 2022): the same words in a different order mean a different image, and models fail.
- **Counting**: Paiss et al., *Teaching CLIP to Count to Ten* (ICCV 2023), with the CountBench benchmark.
- **Spatial reasoning**: *What's "up" with vision-language models?* (Kamath et al., EMNLP 2023). Models are weak on left/right/above/below.

These give me a ready-made **taxonomy of failure types** for the query set: colour, attribute binding, counting, spatial relation, rare word, action, negation.

---

## C. For system 3: video–language models and agents

### Video–language models (the "describer" and the "re-ranker")
- **Qwen2.5-VL** (Bai et al., 2025): the small VLM AVA uses (7B). It handles video, grounding and long context. Newer Qwen VL versions should be checked too.
- **LLaVA-Video / LLaVA-OneVision** (Zhang et al., 2024; Li et al., 2024): open video VLMs.
- **InternVL 2.5 / 3** (Chen et al., 2024/2025): strong open VLMs.
- **Perception Language Model (PLM)** (Cho et al., 2025): Meta's open VLM built on PE Lang, which fits naturally with the PE backbone.

### Long-video agents and video RAG (AVA's baselines)
- **VideoAgent** (Wang et al., ECCV 2024): an LLM agent that decides which frames to look at next.
- **VideoTree** (Wang et al., 2024): a hierarchical, query-adaptive tree of clips for long-video QA.
- **DrVideo** (Ma et al., 2024): turns a long video into a document and reasons over it.
- **Video-RAG** (Luo et al., 2024) and **VideoRAG** (Ren et al., 2025): retrieval-augmented generation over long videos.

### Long-video benchmarks
- **LVBench** (Wang et al., 2024): extremely long videos with QA. AVA evaluates on it.
- **Video-MME** (Fu et al., CVPR 2025): a multi-duration benchmark, and its "long" split.

### Industry systems worth knowing
- **NVIDIA Video Search and Summarization (VSS) blueprint / Metropolis**: the same pattern as AVA (a VLM captions chunks → a vector and graph database → an LLM answers), aimed at CCTV. It's a good sanity check that the industry is going this way.

---

## D. Datasets with fixed cameras (to develop before Whiteshield data arrives, and for public reproducibility)

| Dataset | What it is | Why useful |
|---------|-----------|------------|
| **VIRAT Video** (Oh et al., CVPR 2011) | Outdoor fixed cameras (parking lots, buildings), people and vehicle activities | Closest classic "real CCTV" dataset |
| **MEVA** (Corona et al., WACV 2021) | Multi-camera, about 9,300 h of a staged facility, with activity labels | Many fixed cameras, long durations. Close to my setting. |
| **UCF-Crime** (Sultani et al., CVPR 2018) | About 1,900 long surveillance videos, anomaly labels | Real CCTV quality |
| **UCA: UCF-Crime Annotation** (Yuan et al., CVPR 2024) | UCF-Crime with **time-stamped sentence descriptions** | Already a text-to-surveillance-video grounding benchmark |
| **CityFlow-NL** | Traffic cameras + vehicle descriptions | Vehicle queries |
| **MOT17/20, DukeMTMC (withdrawn), Market-1501** | Tracking / re-ID | Duke was withdrawn for privacy reasons, which is a lesson for the ethics chapter |

---

## E. Query understanding and routing (objective 4)

- **LLM query parsing / text-to-SQL**: turn "white van at the loading bay yesterday after 6" into `{class: vehicle, type: van, colour: white, zone: loading_bay, t >= ...}` plus a leftover free-text part.
- **Model routing / cascades**: *FrugalGPT* (Chen et al., 2023) and *RouteLLM* (Ong et al., 2024) send each query to the cheapest model likely to succeed. This is the same idea as my "router" across the three systems.
- **Query performance prediction** (an information-retrieval idea): predict, before or after retrieving, whether a query will succeed, for example from the score distribution of the top results.
- **Cascaded video analytics** from the systems community: **NoScope** (Kang et al., VLDB 2017), **Focus** (Hsieh et al., OSDI 2018), **Chameleon** (Jiang et al., SIGCOMM 2018). They use cheap models first and expensive ones only when needed, and do cost-accuracy trade-offs over large video archives. **Focus** in particular builds an index at ingest time with a cheap CNN and runs an expensive CNN at query time. That's the same split as my systems 1/2 vs 3, from before VLMs existed.
