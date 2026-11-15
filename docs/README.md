# Reading notes

These are my notes on the papers behind the thesis. I wrote them in plain words, the way I would explain them to someone at lunch. Each note has the same parts: what the paper does, how it works, what matters for my thesis, and what to watch out for.

## Reading order

| # | Note | Paper | Why I read it |
|---|------|-------|---------------|
| 0 | [00_clip.md](00_clip.md) | Radford et al., 2021. CLIP | Background. Everything in system 2 grows out of this. |
| 1 | [01_perception_encoder.md](01_perception_encoder.md) | Bolya et al., 2025. Perception Encoder | Backbone of the embedding system (system 2). |
| 2 | [02_ham_text_to_person_reid.md](02_ham_text_to_person_reid.md) | Jiang et al., CVPR 2025. Modeling Thousands of Human Annotators | Closest research task to "person in a red jacket". |
| 3 | [03_ava_agentic_video_analytics.md](03_ava_agentic_video_analytics.md) | Yan et al., NSDI 2026. AVA | Blueprint for the video–language system (system 3). |
| 4 | [04_faiss.md](04_faiss.md) | Douze et al., 2024/2025. The Faiss Library | Cost, speed and memory of vector search, as the archive grows. |
| 5 | [05_other_relevant_work.md](05_other_relevant_work.md) | Several | Attributes, tracking, datasets, VLMs, known CLIP weaknesses. |
| 6 | [06_privacy_and_eu_ai_act.md](06_privacy_and_eu_ai_act.md) | EU AI Act, GDPR | What the system may and may not do. |

## The thesis in one paragraph

After something happens, someone has to find it in the recordings. Right now that means watching hours of video, or searching the few labels a detector saves ("person", "car"). I compare three ways of searching footage by plain description:

1. **Attribute index.** Save a fixed list of facts about each tracked object, such as colour of top, bag yes or no, vehicle type. It is fast and cheap, but blind to anything not on the list.
2. **Embedding index.** Turn frames and queries into vectors in one shared space (CLIP / Perception Encoder) and look for nearest neighbours. Any wording works, but it struggles with counting, positions and fine colours.
3. **Video–language system.** A small VLM describes the footage as it comes in and stores the descriptions as events. At query time a bigger model re-checks a short list of candidates. It understands the most, and it costs the most.

The question is which one, or which mix, gives good enough answers at a cost and speed I can live with when the archive grows from days to months of footage.

> Note on accuracy: numbers quoted in these notes come from the papers' abstracts and main text. Where I wasn't able to confirm a detail against the paper, I mark it **(to check)**. Before quoting anything in the thesis, go back to the PDF.
