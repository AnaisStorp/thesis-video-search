# Perception Encoder (PE)

**Bolya, D. et al. (2025). Perception Encoder: The best visual embeddings are not at the output of the network. arXiv:2504.13181.** Meta FAIR. Code and weights: `facebookresearch/perception_models`.

## In one sentence

Meta trained a very strong CLIP-style model on images and videos, and found that its best general-purpose features sit in the middle layers of the network, not at the end.

## What they did

1. **Scaled up CLIP training carefully.** They worked through a list of training improvements (resolution schedule, large batches, better optimiser settings, data augmentation and so on) and trained a family of image–text contrastive models. The result is **PE Core**, which comes in several sizes (B, L, G; the G model has about 2B parameters).
2. **Added video.** Good video–caption data is rare, so they built a "video data engine": a captioning model writes detailed captions for videos, and those captions train the video side. They also released a video dataset made this way (the PE Video Dataset, PVD) **(size: to check)**. The video embedding is basically frame embeddings averaged over time.
3. **The main finding (the title).** For tasks that aren't retrieval, such as detection, tracking, depth or feeding a chat model, the most useful features are in **intermediate layers**. The last layer is very good at "image ↔ text" matching but throws away spatial detail.
4. So they released two extra versions that bring those middle-layer features out to the output:
   - **PE Lang**: tuned to be plugged into a multimodal LLM as its vision encoder.
   - **PE Spatial**: tuned for dense tasks such as detection, segmentation and tracking.

## Results, roughly

PE Core was state of the art (at release) on zero-shot image classification and image–text and video–text retrieval, and it holds up well on harder, "robust" image sets. PE Lang and PE Spatial are competitive with or better than the specialist models in their areas. Exact numbers: see tables 1–3 of the paper.

## Why it matters for my thesis

- **Backbone for system 2.** Use PE Core to embed (a) person or vehicle crops from the tracker and (b) whole frames or short clips. Embed the text query with PE's text encoder. Then search with Faiss.
- **Crops vs frames vs clips** is a real design choice I should test:
  - Crops are good for appearance ("red jacket").
  - Frames are good for scene context ("near the loading bay").
  - Clips are good for actions ("running", "fighting").
- **Model size is a cost knob.** B vs L vs G changes indexing GPU time per hour of footage. That's one axis of my cost comparison.
- **PE Spatial** could also feed the tracker or detector in system 1. That's optional.
- **PE Lang** is relevant if I build my own VLM for system 3. It's Meta's encoder for the Perception Language Model (PLM).

## Things to watch

- The PE models are trained on web data, so CCTV is out of domain. Measure that, don't assume it.
- It's still a contrastive model, so I expect the same weaknesses as CLIP on counting and positions. I should check this rather than assume it improved.
- Check the licence before using it in a company setting.
- Embedding dimension (it varies by model size; L is around 1024 **(to check)**) drives memory in Faiss. See note 04.
