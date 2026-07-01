---
title: A Generative Paradigm of CTR Prediction Models
date: 2026-03-01 10:00:00 +0800
categories: [Paper, CTR]
tags: [supervised_feature_generation]
render_with_liquid: false
math: true
---

# From Feature Interaction to Feature Generation: A Generative Paradigm of CTR Prediction Models

- **arXiv:** [2512.14041](https://arxiv.org/abs/2512.14041)
- **Authors:** Mingjia Yin, Junwei Pan, Hao Wang, Ximei Wang, Shangyu Zhang, Jie Jiang, Defu Lian, Enhong Chen (USTC & Tencent)
- **Venue:** ICML 2025 style; deployed at Tencent
- **Code:** https://github.com/USTC-StarTeam/GE4Rec

## TL;DR

CTR models today follow a *discriminative* paradigm that repeatedly multiplies raw ID embeddings against each other. This causes (1) *dimensional collapse* (embeddings live in a low-dim subspace) and (2) *information redundancy* (the two "views" being interacted are strongly correlated). The paper proposes **Supervised Feature Generation (SFG)**: a general Encoder–Decoder wrapper that reformulates almost any existing CTR model into a generative "feature generation" paradigm, while keeping the supervised click label as the training signal. A tiny field-wise non-linear MLP encoder + the base model's interaction matrix as decoder gives a consistent AUC lift across FM, DeepFM, IPNN, xDeepFM, FmFM, CrossNet, and DCN V2, and drove a **+2.68% GMV** win when deployed to Tencent's ad platform.

## Motivation

- Existing CTR models compute $\mathcal{L}_{\text{BCE}}(y_{\text{sup}}, f_{\text{cls}}(g_{\text{inter}}(\{\bm v_i\})))$: features are just IDs, so raw embeddings $\bm v_i$ are directly multiplied against each other.
- Three issues follow:
  1. **Dimensional collapse** — Guo et al.'s *Interaction-Collapse Theory*: low-cardinality fields drag other fields into a low-rank subspace.
  2. **Only learning $P(\mathcal{Y}\mid\mathcal{X})$, ignoring $P(\mathcal{X})$**, so the model never captures the rich co-occurrence structure of the input.
  3. **Redundancy** — the two operands in every interaction are the same raw embeddings, so they carry highly correlated information (violates the Barlow-Twins-style redundancy-reduction principle).
- Sequential recommenders escape this by using next-item prediction, but tabular CTR data has no global partial order. VAR-style "reconsider the order" thinking motivates asking: *what is the inherent structure of multi-field categorical data?* Answer: **feature co-occurrence**.

## Method: Supervised Feature Generation (SFG)

### Paradigm

Treat one side of a co-occurring feature pair as `x_source` and the other as `x_target`, and let an Encoder–Decoder generate `x_target` from `x_source`. Because there is no partial order, both sides are the full feature set — an **"All-Predict-All"** scheme.

$$
\mathcal{L}\Big(y_{\text{sup}},\; f_{\text{cls}}\big(f_{\text{decoder}_{i\to j}}\big(f_{\text{encoder}_i}(\{\bm v_k\}_{k=1}^N)\big),\; \bm v_j\big)\Big)
$$

- **Encoder** $f_{\text{encoder}_i}$ builds a *new* per-feature representation $\bm z_i$ from all raw embeddings.
- **Decoder** $f_{\text{decoder}_{i\to j}}$ maps $\bm z_i$ back to the space of feature $j$.
- **Classifier** $f_{\text{cls}}$ scores the (generated, target) pair, pools across all $(i,j)$, and gives $\hat y$.
- **Loss** is the ordinary supervised BCE on the click label — not a self-supervised "predict-the-masked-feature" loss. Self-supervised generative losses would leak the label because the "true other feature" is already in `x_source`.

### Architecture

- **Encoder (kept intentionally minimal):** field-wise single-layer non-linear MLP
  $$f_{\text{encoder}_i}([\bm v]) = \sigma\big([\{\bm v_k\}_{k=1}^N]\, W_{F(i)}\big),$$
  with ReLU $\sigma$ and a **field-specific** projection $W_{F(i)}\in\mathbb R^{Nd\times d}$. Ablations show every element (concat-all, non-linearity, field-specific weight, one layer) is needed — simpler hurts, deeper overfits.
- **Decoder:** just the interaction matrix of whatever base model you plug in. Different $\bm W$ shapes recover different classics:
  - scalar $w_{F(i)\to F(j)}\mathcal{I}$ ⇒ FwFM / xDeepFM
  - full $\bm W_{F(i)\to F(j)}$ ⇒ FmFM / DCN V2
- **Example (generative FmFM):**
  $$\mathcal{L}\Big(y,\; \sum_{i,j}\big(\sigma([\{\bm v_k\}]\,W_{F(i)})\,W_{F(i)\to F(j)}\big)\odot \bm v_j\Big).$$
- Multiple layers are supported: layer $\ell$'s output becomes both `x_source` and `x_target` at layer $\ell{+}1$.

### Why it fixes the two pathologies

- **No direct raw-embedding product.** The encoder inserts a non-linear, all-fields-mixed projection before the interaction, so low-cardinality fields no longer drag others down (mitigates dimensional collapse).
- **Encoder output is sample-specific and decorrelated from raw embeddings**, satisfying the redundancy-reduction principle.

### Relationship to prior gating tricks

SFG generalises SENet (FiBiNET), PEPNet's GateNU, and FinalMLP's feature-gating layer: those methods can be read as encoders producing gating coefficients; SFG reinterprets them as *representation learners* inside a generative framework.

## Experiments

### Datasets & protocol

- Criteo and Avazu, evaluated with AUC and Logloss (FuxiCTR library).
- Base models plugged into SFG: FM, FmFM, CrossNet, DeepFM, IPNN, xDeepFM, DCN V2.

### RQ1 — Does it lift performance?

- **Average +0.272% AUC / −0.435% Logloss** over discriminative baselines (0.1% AUC is considered a big lift in this field).
- Explicit-interaction models benefit most (+0.428% AUC / −0.689% Logloss on average).
- Notable: **generative CrossNet beats discriminative DCN V2** (+0.106% AUC), even though DCN V2 has an extra DNN branch.
- The paradigm shift **narrows the gap between weak and strong architectures** (Criteo: FM–DCNv2 gap shrinks from 1.151% AUC to 0.364%).
- Cost is small: +3.14% compute time, +1.45% GPU memory on average.

### Online A/B at Tencent

- Deployed by swapping the IPNN expert inside a Heterogeneous-Experts + Multi-Embedding production model (500+ features).
- One-week 20% A/B: **+2.68% GMV, +2.46% CTR** across Moments pCTR, Content & Platform pCTR, and DSP pCTR; now the production model. Described as one of the largest revenue lifts at Tencent in 2024.

### RQ2 — Why does it work?

- **Dimensional collapse (SVD of embedding covariance):** Baseline DCN V2 on Criteo shows singular values dropping ~10¹⁰× around index 250 — roughly 30% of dimensions are dead. SFG produces smooth, slowly-decaying spectra across FM, DeepFM, CrossNet, DCN V2; even FM gains ~25% more usable dimensions.
- **Redundancy (Pearson correlation between the two interacted vectors):** Discriminative DCN V2 still shows visible intra-field diagonal blocks; **generative DCN V2's correlation matrix is essentially zero**. The paper also documents a clear negative correlation between the redundancy metric and offline AUC across FM → DeepFM → DCN V2.

### RQ3 — Ablations on framework design (on DCN V2)

- **`x_source`:** using all fields as source > using only own field, and using all fields *for low-cardinality targets* helps more than for high-cardinality ones — matching the collapse story.
- **Encoder:** field-wise > field-shared; non-linearity is essential; adding a second layer hurts (overfits); replacing MLP with self-attention beats the discriminative baseline but loses to the MLP encoder.
- **`x_target` / generation scheme:** "Predict-All" beats predict-random and beats masked feature modeling (MFM). Vanilla MFM with a single learnable mask *underperforms* the baseline; field-aware masks or a zero mask recover most of the gain but still trail Predict-All. Learnable masks + supervised signal apparently conflict.

## Takeaways

- Reframing CTR from "interact raw embeddings" to "generate all features from all features" is a drop-in wrapper that fits FM through DCN V2.
- Keep the supervised label — self-supervised losses over the same inputs leak.
- The Encoder must be **field-specific, non-linear, one-layer, and take all embeddings as input**; complexity beyond that hurts.
- The real mechanism is representation quality: SFG buys you non-collapsed, decorrelated feature embeddings, and that translates to consistent offline lifts and a very large online GMV lift.

## Related Work in One Line

Orthogonal to prior effort spent on ever-fancier interaction functions (FM, FwFM, FmFM, xDeepFM, DCN V2/V3) and gating-based non-linearity (FiBiNET, PEPNet, FinalMLP); this paper changes the *paradigm* rather than the interaction operator.
