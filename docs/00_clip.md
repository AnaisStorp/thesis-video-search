# CLIP: the background for everything

**Radford, A. et al. (2021). Learning Transferable Visual Models From Natural Language Supervision. ICML 2021.** (arXiv:2103.00020)

## In one sentence

OpenAI took about 400 million image–caption pairs from the internet and trained an image encoder and a text encoder so that an image and its caption land close together in the same vector space.

## How it works, simply

- There are two networks: one reads images (a ResNet or a Vision Transformer), and one reads text (a Transformer).
- Each training batch holds N images and their N captions. The model gets N×N possible pairings and must pick the N correct ones. This is **contrastive learning**: pull matching pairs together and push wrong pairs apart.
- Afterwards you can compare any image with any sentence by taking the cosine similarity of their vectors.

## Why it was a big deal

- **Zero-shot.** To classify an image you don't need to train a classifier. You write "a photo of a dog", "a photo of a cat" and so on, and pick the closest one. On ImageNet this matched a supervised ResNet-50 without seeing a single ImageNet label.
- **Open vocabulary.** The search terms don't have to be decided in advance. This is exactly what my fixed detector can't do.

## Why it matters for my thesis

System 2 (the embedding index) is basically CLIP-style search. You embed frames or object crops once, store the vectors, then embed the operator's question and look up the nearest vectors. Perception Encoder (note 01) is a modern, stronger version of the same idea.

## Known weaknesses (these are my hypotheses for objective 4)

- **Bag of words.** CLIP doesn't really read word order or structure. "A man in a red shirt next to a woman in blue" and "a woman in a red shirt next to a man in blue" end up almost identical (see ARO and Winoground in note 05).
- **Counting.** "Three people" and "two people" are hard to tell apart.
- **Positions.** "Left of", "behind", "near the gate" are weak.
- **Small objects and fine colour.** In a wide surveillance shot a person is maybe 50×120 pixels. CLIP sees the whole frame at 224 or 336 px, so the person's jacket is a few pixels. **Cropping each tracked person before embedding is probably essential.**
- **Domain gap.** It was trained on web photos (centred, well lit, high resolution), not on grainy top-down CCTV at night.
