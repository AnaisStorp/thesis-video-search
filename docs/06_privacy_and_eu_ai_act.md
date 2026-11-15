# Privacy, the EU AI Act and GDPR: what it means for this system

> I'm not a lawyer. This is my reading to frame the design. The company's DPO or legal team should confirm it.

## Why this matters

Searching for "a person in a red jacket" across all cameras and all days is very close to **tracking an individual**. The same tool that finds a thief can follow an employee. The thesis has to show the system is designed to stay on the right side of that line.

## EU AI Act (Regulation (EU) 2024/1689), the parts that matter

- **Remote biometric identification (RBI)** means identifying people at a distance by comparing **biometric data** (face, gait, etc.) with a reference database.
  - *Real-time* RBI in public spaces for law enforcement is **prohibited** except in narrow cases (Art. 5).
  - *Post* (retrospective) RBI is classed as **high-risk** (Annex III), with heavy obligations.
- **Biometric data** means data from specific technical processing of physical, physiological or behavioural traits that **allows or confirms unique identification**.
- Searching by **clothing and context** ("red jacket", "carrying a box", "near gate 3") is *not*, by itself, biometric identification, because a jacket doesn't uniquely identify you the way a face does. This is the key design argument. It stays valid only if:
  - there is **no face recognition** and no face embeddings are stored. Crops should ideally have the face **blurred** before embedding, so the embedding can't secretly act as a face template;
  - there is **no gait-based matching**;
  - there is **no "find this same person everywhere"** feature that links tracks across cameras into an identity (out of scope anyway).
- **Biometric categorisation** that infers sensitive traits (ethnicity, religion, etc.) is prohibited (Art. 5). So the **attribute list for system 1 must not include those**. Even "gender" and "age" are borderline, so discuss them and maybe drop them.
- **Transparency / logging**: high-risk systems need automatic logging. Even if this system isn't classed high-risk, keeping an **audit log** (who searched, what query, when, which results were opened) is good practice and is required by the subject.

## GDPR (CCTV footage is personal data)

- **Purpose limitation**: footage collected for security can be used for incident investigation. Is "thesis research" covered? → Needs a check and probably a **DPIA** (data protection impact assessment) plus a data access agreement.
- **Data minimisation and retention**: the index (descriptions, embeddings) is also personal data. **It must not outlive the footage's retention period.** If video is kept 30 days, so is the index. That limits my "months of footage" scaling experiments on real data. I might need to simulate long archives (e.g. replay, or use public datasets for scale).
- **Security**: footage stays on company infrastructure. **No sending frames to external APIs** (GPT-4o, Gemini) unless legally cleared. This decides which VLMs I can use in system 3 → open models on local GPUs.

## Design rules I'll follow (to put in the thesis)

1. No face recognition, no face embeddings, faces blurred in stored crops.
2. No sensitive-trait attributes.
3. No cross-camera identity linking.
4. Every query is logged (user, time, query text, filters, results viewed).
5. The index follows the same retention as the video.
6. Everything runs on-premises.
7. Queries should be tied to an incident or case ID (purpose binding). That's optional, but a good idea.
