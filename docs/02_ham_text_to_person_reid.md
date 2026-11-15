# Modeling Thousands of Human Annotators (text-to-image person re-identification)

**Jiang, J. et al. (2025). Modeling Thousands of Human Annotators for Generalizable Text-to-Image Person Re-identification. CVPR 2025.**

> ⚠️ I couldn't automatically fetch this paper while writing these notes. The ideas below are what I know of it. **Check every name and number against the PDF** before citing.

## The task first: text-to-image person search (also called text-based person re-identification)

You have a gallery of cropped pedestrian images and a sentence like "a young woman with long black hair, wearing a white T-shirt, blue jeans and carrying a black backpack". You must rank the gallery so that the matching person comes first. Standard datasets:

- **CUHK-PEDES** (Li et al., 2017): about 40k images of about 13k people, two descriptions per image. This is the reference benchmark.
- **ICFG-PEDES** (Ding et al., 2021): longer and more detailed descriptions.
- **RSTPReid** (Zhu et al., 2021): images from more cameras, closer to real surveillance.

Metric: Rank-1 / Rank-5 / Rank-10 and mAP. This is the same idea as my recall@k.

## The problem this paper attacks

Models trained on one of these datasets **do badly on the others**. One big reason is that the descriptions in each dataset were written by a small group of annotators with their own habits: the same order of mentioning things, the same vocabulary. The model learns those habits instead of learning to read any description.

Collecting more human descriptions is expensive. So people use multimodal LLMs (MLLMs) to caption large unlabelled person datasets, such as LUPerson. But MLLM captions all sound the same, so you've just swapped a few human styles for one robot style.

## The idea

**Make the MLLM write like thousands of different humans.**

1. Take the existing human-written descriptions and pull out a "style" representation for each one (how it's written, not what it says).
2. **Cluster** those styles into a large number of prototypes. Each prototype acts as one "virtual annotator".
3. When captioning a new image, pick a style prototype and prompt the MLLM to describe the person *in that style*.
4. Sample prototypes so the styles are covered **evenly**, including rare ones, not just the common ones (a "uniform prototype sampling" idea).
5. This gives a big, stylistically varied synthetic dataset. Pre-train a text–person retrieval model on it.

## Result

A model pre-trained on this synthetic data generalises much better to CUHK-PEDES, ICFG-PEDES and RSTPReid. That includes direct transfer, where it's tested on a dataset it never trained on. Fine-tuning on top gives state-of-the-art results. (Exact numbers: see the paper.)

## Why it matters for my thesis

- **It's the closest research task to "a person in a red jacket".** The difference is that it works on clean, already-cropped photos, while I work on full camera views with time and place. A specialist text–person model like this could be a **fourth option** or a stronger version of system 2 just for person queries.
- **The real lesson is about queries.** Operators will word things in many different ways ("red coat", "guy in a red top", "someone wearing red"). My **test set must include many phrasing styles**, otherwise I'm only testing my own way of writing. I could use the same trick: have an LLM rewrite each test query in several styles and check if recall holds.
- **Synthetic captions as cheap labels.** The same pipeline (MLLM writes descriptions of tracked people) is basically how I'd build attribute labels or extra training data from company footage.

## Limits for my setting

- The crops are clean, mostly frontal and high resolution. My footage is low resolution, at odd angles, partly hidden and at night.
- There's no time, no place and no action, just appearance.
- Description styles come from academic annotators, not security operators.
