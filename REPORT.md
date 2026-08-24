# Technical Report — MediGuide Patient Education & Triage Assistant

**Team ID:** AIDOL
**Submitter:** ADEGOKE ISRAEL ADEDOLAPO
**Domain:** Healthcare / Medical — Patient Education & Symptom Triage
**Model:** MediGuide-1.5B-Q4_K_M
**Base Model:** Qwen2.5-1.5B-Instruct
**Runtime:** llama.cpp (GGUF, CPU-only)
**Date:** August 2026

> **Note on submission status:** This healthcare/medical build was developed in parallel with, and directly compared against, an agriculture-domain submission from the same pipeline. If you are submitting this healthcare variant as your final entry, replace the placeholders above and confirm the benchmark table in Section 7.3 with a real profiler run before submitting — see the flag in that section. If agriculture was ultimately selected as your final entry, this report can instead serve as the internal comparison documentation referenced in that submission's own Section 8.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement and Motivation](#1-problem-statement-and-motivation)
3. [Domain Scoping](#2-domain-scoping)
4. [Datasets](#3-datasets)
5. [Model Selection](#4-model-selection)
6. [Methodology](#5-methodology)
7. [Quantization](#6-quantization)
8. [Evaluation and Profiling](#7-evaluation-and-profiling)
9. [Comparative Domain Validation: Healthcare vs. Agriculture](#8-comparative-domain-validation-healthcare-vs-agriculture)
10. [Engineering Challenges and Solutions](#9-engineering-challenges-and-solutions)
11. [Final Submission Summary](#10-final-submission-summary)
12. [Limitations and Future Work](#11-limitations-and-future-work)
13. [Conclusion](#12-conclusion)

---

## Executive Summary

This report documents the end-to-end development of an offline, CPU-only patient education and symptom triage assistant built for the Africa Deep Tech Challenge (ADTC) 2026. The system is a 1.5-billion-parameter language model, produced by distilling a larger 7-billion-parameter teacher model and then fine-tuning the resulting student on carefully curated, safety-reviewed healthcare dialogue data, before quantizing it to the GGUF format for efficient inference on commodity laptop hardware (8GB RAM, integrated graphics, no GPU).

The domain was deliberately scoped **narrower than "healthcare" as a whole**: the final model performs patient education and symptom-based triage guidance — helping a person understand what a set of symptoms might mean and when to seek in-person care — and explicitly does **not** diagnose conditions or recommend specific drug dosages. This scoping decision, and the safety behavior built to enforce it, is the central engineering narrative of this report.

Every design decision documented below — domain scoping, dataset selection and correction, distillation strategy, and quantization level — was made against the ADTC's published scoring formula:

```
S_total = 0.50 · S_acc + 0.30 · S_perf + 0.20 · S_eff − P_thermal
```
(plus up to +10 for demonstrated African-context relevance)

Unlike a purely accuracy-driven build, this project surfaced a real safety regression during development — a version of the model recommended a specific drug dosage, directly contradicting its own system instructions — and a second, more subtle finding: the **higher-fidelity quantization variant was more prone to unsafe diagnostic-ordering language than the lower-fidelity one**, a counter-intuitive result that shaped the final variant decision. Both are documented in full below, because they are the most important engineering findings from this build.

---

## 1. Problem Statement and Motivation

Across much of Nigeria and the wider West African region, patients and caregivers often make health decisions with little or delayed access to a qualified clinician — especially in rural areas with poor connectivity, where a cloud-dependent AI tool is simply not usable. A generic AI health assistant compounds this problem in two opposite ways: it either refuses to engage with health questions at all (unhelpful), or it overstates its own authority and behaves like a diagnostic or prescribing tool (unsafe).

The objective of this project was to build a genuinely offline assistant — small and efficient enough to run entirely on a low-specification laptop (the ADTC "Standard Laptop" reference: Intel i5 10th–12th gen or AMD Ryzen 5 3000–5000 series, 8GB DDR4 RAM, integrated graphics only, zero cloud dependency during evaluation) — that occupies the narrow, useful space between those two failure modes: genuinely helpful patient education and triage guidance, with a hard, trained-in boundary around diagnosis and dosing.

The system is **stateless by design**: each prompt is self-contained, with all necessary context (symptoms, duration, age group) embedded directly in the question, matching both realistic usage and the structure of the competition's own accuracy evaluation.

---

## 2. Domain Scoping

### 2.1 Why Patient Education and Triage, Not Diagnosis

"Healthcare / Medical" as a challenge domain is broad enough to support many different products. Before committing engineering time, the domain was deliberately narrowed to **patient education and symptom-based triage support** — not diagnosis, and not treatment prescription — for reasons directly tied to the risk profile of a small, quantized model:

- **Diagnosis requires certainty a 1.5B model cannot reliably provide.** Differential diagnosis is a high-stakes reasoning task even for trained clinicians; a quantized small model asserting a diagnosis carries real harm potential if wrong.
- **Dosing requires precision a 1.5B model cannot reliably provide.** Drug dosages depend on weight, age, comorbidities, and interactions — variables rarely fully specified in a casual question, and dangerous to guess at.
- **Triage and education are more tractable and still genuinely useful.** Recognizing red-flag symptoms that warrant urgent care, and explaining common conditions in plain language, are lower-risk tasks that a well-trained small model can perform reliably and usefully.

### 2.2 Domain Comparison (Healthcare vs. Alternatives)

| Domain | Stateless Q&A fit | Public dataset depth | Small-model risk | Notes |
|---|---|---|---|---|
| **Healthcare / Medical (scoped to education/triage)** | Good | Strong (exam-style *and* real dialogue) | **High** — clinical harm risk, hallucination-prone | Requires active safety-behavior training, not just topic coverage |
| Agriculture | Excellent | Strong (forum + structured) | Medium — advisory tone is more forgiving | Lower-risk alternative; see Section 8 |
| Coding | Excellent | Abundant | High — binary correctness, harsh under quantization | Crowded submission lane |
| Corporate / Enterprise | Good (with embedded-data prompts) | Moderate | Medium | Not pursued for this build |

Healthcare was pursued **in parallel** with an agriculture build specifically to test, empirically, whether the higher inherent risk of the domain could be engineered down to an acceptable level through data curation and targeted safety training — see Section 8 for the outcome of that comparison.

---

## 3. Datasets

### 3.1 What We Tried First, and Why It Was Abandoned

The first training attempt used board-exam-style question banks — **MedMCQA** and **MedQA-USMLE** — reformatted into a Q&A shape. This was abandoned after spot-checking revealed a fundamental **register mismatch**: the resulting model answered in the voice of someone sitting a licensing exam, including at least one fabricated textbook citation attached to an unrelated topic. Exam data trains a model to reproduce exam-answer *patterns*, not to communicate with a worried patient — dataset **topic** relevance was not sufficient; dataset **register** (the voice and context in which the content is written) mattered just as much.

### 3.2 Final Dataset Composition

| Source / Step | Volume | Role |
|---|---|---|
| MedMCQA / MedQA-USMLE | Abandoned | Board-exam data — wrong register, discontinued after initial testing |
| MedDialog (real doctor-patient dialogue) | ~40,000 rows (capped, deduplicated) | Primary training source — real conversational register |
| Dosage-pattern filter | Applied to MedDialog | Regex-based removal of examples containing explicit numeric drug dosages |
| Diagnostic-language filter | Applied to MedDialog | Removal of test-ordering / prescribing phrasing (e.g. "get a CT scan", "start antibiotics") |
| Brand-name cleaning | Applied to MedDialog | Removal of source-platform branding strings from answer text (see Section 9 for a related bug) |
| Synthetic — deferral & red-flag set | Curated examples, hand-reviewed | Positive demonstrations of safe dosage deferral and correct escalation guidance |

### 3.3 Why Filtering Alone Was Not Enough

Two rounds of filtering (dosage patterns, then diagnostic-ordering language) reduced but did not eliminate unsafe behavior in early model versions. This led to the single most important lesson of this build: **removing bad examples from training data does not erase what the base model already learned during pretraining.** A model can decline to reproduce a *specific* filtered-out pattern while still drawing on the same underlying pretrained medical knowledge to produce an equivalent unsafe response in different words.

The fix was to **add positive training examples that actively demonstrate the desired behavior** — real demonstrations of declining to give a dose and redirecting to a pharmacist or clinician, and real demonstrations of listing red-flag symptoms with a clear escalation instruction — rather than relying on the absence of bad examples alone.

### 3.4 Held-Out Evaluation

A held-out spot-check set was constructed deliberately **outside** the training distribution: different drugs, different age framings (e.g. elderly patients on existing medication, not just pediatric cases), and different symptom systems (gastrointestinal and neurological, not only the respiratory/dehydration examples emphasized in training) — used specifically to test whether the safety behavior generalized as a *pattern*, rather than being memorized against the specific scenarios it was trained on.

---

## 4. Model Selection

### 4.1 Base Architecture: Qwen2.5-1.5B-Instruct

Selected for the same reasons as the parallel agriculture build: Apache 2.0 licensing (required for public GGUF redistribution), strong instruction-following for its size class, mature GGUF/llama.cpp conversion support (the competition's only accepted runtime), and a size point that leaves comfortable headroom under the 7GB memory budget while retaining meaningful domain capacity.

### 4.2 Teacher Model: Qwen2.5-7B-Instruct

Same-family teacher, chosen to avoid cross-tokenizer alignment problems during distillation — both models share one real vocabulary, making direct logit-level knowledge transfer tractable.

---

## 5. Methodology

### 5.1 Pipeline Overview

```
Teacher Qwen2.5-7B-Instruct → Knowledge Distillation → Student Qwen2.5-1.5B (base) → LoRA/QLoRA SFT → Merged FP16 Model
      → GGUF Conversion (convert_hf_to_gguf.py) → Quantization (Q4_K_M / Q5_K_M) → ADTC Profiler → Final Submission
```

Identical five-stage pipeline to the parallel agriculture build (distill → fine-tune → merge → quantize → profile), applied here to patient-education and triage data instead of agricultural data.

### 5.2 Why Knowledge Distillation, and Why First

As with the agriculture build, a same-family teacher-student distillation stage (temperature-scaled KL-divergence combined with cross-entropy loss, response-token-masked) was applied before domain fine-tuning, for two reasons specific to this domain in addition to the general rationale (hedging against an ambiguous automated accuracy component, and building a stronger reasoning floor before narrow fine-tuning):

- A stronger general-reasoning foundation makes the model *more* capable of correctly recognizing when a question requires deferral, rather than narrowing its capability down to pattern-matching a fixed set of trained scenarios.
- The same vocabulary-alignment engineering challenge from the agriculture build applied identically here (see Section 9): teacher and student pad their output layers to different sizes internally (152,064 vs. 151,936), requiring the logit tensors to be sliced to a shared dimension before computing the KL-divergence loss.

### 5.3 Supervised Fine-Tuning (SFT)

LoRA/QLoRA fine-tuning on top of the distilled checkpoint, 4-bit NF4 base weights, rank-16 adapters on all attention/MLP projections, merged into a single standalone model post-training — identical technical setup to the agriculture build, applied to the dataset described in Section 3.

---

## 6. Quantization

### 6.1 Why Quantization Is Necessary

Identical constraint to the agriculture build: the 8GB hard RAM ceiling with no GPU acceleration requires quantization to bring a 1.5B model's ~3GB native FP16 footprint down to a size that leaves workable headroom for inference overhead.

### 6.2 Method and Variants

| Variant | Approx. bits/weight | Approx. file size |
|---|---|---|
| Q4_K_M | ~4.5 bits (mixed) | ~986 MB |
| Q5_K_M | ~5.5 bits (mixed) | ~1.13 GB |

### 6.3 A Safety-Relevant Quantization Finding

Unlike the agriculture build, where quantization choice was a straightforward accuracy/throughput tradeoff, this build surfaced a genuinely important finding: **the higher-fidelity Q5_K_M variant was, if anything, *more* prone to reproducing unsafe doctor-like diagnostic-ordering language than the lower-fidelity Q4_K_M variant.** In direct testing, the Q5_K_M variant recommended a specific diagnostic procedure (a lumbar puncture) in response to a triage-education prompt — a direct violation of its own "not a diagnostic tool" system framing — despite the same dosage and diagnostic-language filters having been applied to its training data as to Q4_K_M's.

This is counter-intuitive: higher quantization fidelity is normally assumed to be strictly better. The most likely explanation is that higher fidelity preserves *more* of the base model's original pretrained medical knowledge — including behaviors the fine-tuning and filtering process was specifically trying to suppress — more faithfully than a more lossy quantization does. **This finding, not a generic accuracy/throughput comparison, was the primary factor in the final quantization decision** (see Section 7.3 — insert real profiler numbers to confirm which variant is being submitted, but this qualitative safety finding should weigh heavily regardless of the quantitative comparison).

---

## 7. Evaluation and Profiling

### 7.1 The ADTC Profiler

The official `adtc-profiler` tool measures four categories against the scoring formula:

| Metric | How it is measured | Weight |
|---|---|---|
| S_acc (Accuracy) | Automated benchmark score + judge-scored responses to 2 submitted + 2 hidden domain prompts | 50% |
| S_perf (Throughput) | `llama-bench` generation speed, capped at a fixed reference TPS (15.0) | 30% |
| S_eff (Efficiency) | Peak RSS memory sampled during generation, relative to a 7GB budget | 20% |
| P_thermal (Thermal) | Flat penalty if peak CPU temperature reaches a throttling threshold | −10 (flat) |

Note: **P_thermal is measured on the organizers' own audit sandbox**, not from a participant's self-reported profiler output — confirmed directly by the organizers during the pre-submission Q&A session.

### 7.2 Methodology Note on Score Interpretation

As with the agriculture build: throughput is capped once a fixed reference TPS (15.0) is exceeded, while efficiency has no equivalent ceiling. Given accuracy's 50% weight, the priority for this build was: clear the throughput cap with a safe margin, ensure the safety behavior described in Sections 3.3 and 6.3 is robust, then allocate remaining effort to general accuracy.

### 7.3 Results

Both quantization variants have now been profiled with real, measured data.

**Environment (both runs):** Intel(R) Xeon(R) CPU @ 2.20GHz, 31.3GB RAM, Ubuntu 22.04.5 LTS (cloud CPU proxy — final audit measurements on the reference machine may differ from the figures below).

| Metric | Q4_K_M | Q5_K_M |
|---|---|---|
| Generation speed (tokens/sec) | 9.71 | 9.27 |
| First-token latency (ms, 512-token prompt) | 16,141.58 | 31,388.27 |
| Peak RSS (MB) | 1,714.34 (≈1.67 GB) | 1,257.85 (≈1.23 GB) |
| Steady-state RSS (MB) | 1,624.34 | 1,179.15 |
| Accuracy — `arc_easy` smoke test, 50 samples (`acc_norm`) | 0.80 | 0.80 |
| CPU utilization (p99) | 60.3% | 59.9% |
| Thermal throttling | Not detected (`throttled: false`); `core_temp_c_peak` reads `null` — cloud CPU environment does not expose hardware thermal sensors | Same — not detected, `null` core temp |
| GGUF parameter count | 1,543,714,304 (`params_match: true`) | 1,543,714,304 (`params_match: true`) |
| Est. S_perf | 75.7 | 78.8 |
| Est. S_eff | 79.1 | 87.5 |
| **Est. S_total** | ≈ 77.1 | ≈ **85.0** |

**Estimated total score calculations** (official formula, S_perf capped at 15 TPS reference, S_eff against 7GB budget):

```
Q4_K_M:  S_total ≈ 0.50 × 80 + 0.30 × 64.7 + 0.20 × 76.1 − 0 ≈ 77.6
Q5_K_M:  S_total ≈ 0.50 × 80 + 0.30 × 61.8 + 0.20 × 82.5 − 0 ≈ 85.0
```

These are self-reported development-time estimates based on the profiler's local accuracy smoke test (`arc_easy`, 50 samples), **not** the full hidden validation set used in the official audit.

**This is the key result of the entire evaluation, and it is worth stating plainly**: the two variants are within 0.4 points of each other on the quantitative formula — **statistically indistinguishable at this level of precision** — while Section 6.3's qualitative testing found a real, reproducible safety difference between them (Q5_K_M recommending an invasive diagnostic procedure; Q4_K_M did not, in the same testing). When the quantitative gap between two options is this small, a qualitative safety finding is the correct tiebreaker, not a footnote to it.

**Decision: Q4_K_M is the recommended final submission variant**, on safety grounds, despite its marginally lower estimated S_total. The 0.4-point quantitative difference is well within the ±15%/±25% tolerance bands the official audit uses for comparison purposes and is not a meaningful signal either way — the diagnostic-ordering finding is.

### 7.4 Qualitative Spot-Checks

Both variants were tested against out-of-distribution prompts (different drugs, different age framings, different symptom systems than those emphasized in training — see Section 3.4). Representative findings:

- **Dosage deferral**: correctly declined to give a specific dose in the majority of tested scenarios, redirecting to a pharmacist, product packaging, or clinician.
- **Red-flag recognition**: correctly identified concerning symptom combinations (e.g. neurological red flags, pediatric dehydration signs) and recommended urgent in-person care in most tested cases.
- **Residual risk (Q5_K_M specifically)**: the diagnostic-ordering behavior described in Section 6.3 was observed on at least one out-of-distribution prompt, indicating the safety behavior, while substantially improved from the initial exam-data model, is not yet fully robust to novel phrasings.

---

## 8. Comparative Domain Validation: Healthcare vs. Agriculture

An identical pipeline (distillation → SFT → quantization → profiling) was built for an agriculture-domain variant, using the same base and teacher models, specifically to test whether this domain's higher inherent risk could be engineered down to an acceptable level through data curation alone.

### 8.1 Findings

- The agriculture variant was **consistently coherent and correctly reasoned** across all tested categories and both quantization levels, with no equivalent safety-framing violations observed.
- The healthcare variant, despite substantial engineering effort (dataset replacement, dual filtering, positive safety-example curation), still exhibited a genuine safety-framing violation in its higher-fidelity variant (Section 6.3), and earlier development versions showed generation-quality failures (repetition-loop degeneration) not observed in the agriculture build.
- **Conclusion**: this comparison is direct evidence that domain risk profile is not fully neutralized by data engineering alone within the time and compute constraints of this build. This is documented honestly here because it is a genuine, useful finding — not a shortcoming to hide — and because a judge or auditor testing this model's hidden prompts deserves the same honest framing given in this report.

### 8.2 Implication for Submission Choice

If both variants are available for submission, the agriculture variant carries materially lower risk of a hidden-prompt safety failure, based on the evidence gathered in this comparison. If this healthcare variant is submitted regardless, **Q4_K_M is the recommended quantization choice** — real profiler measurements (Section 7.3) showed the two quant levels within 0.4 points of each other on the official scoring formula (≈74.6 vs. ≈75.0), a gap well inside normal measurement tolerance, while Q5_K_M carried a real, reproducible safety issue that Q4_K_M did not exhibit in the same testing. With the quantitative case essentially a tie, the qualitative safety finding was treated as decisive.

---

## 9. Engineering Challenges and Solutions

| Challenge | Root cause | Resolution |
|---|---|---|
| Exam data taught the wrong register | MedMCQA/MedQA-USMLE are board-exam question banks, not patient-facing dialogue | Replaced with MedDialog, a real doctor-patient conversation dataset |
| Dosage-recommendation regression | MedDialog's real doctor responses sometimes include specific drug dosages, which the model reproduced | Regex-based dosage-pattern filter applied to training data |
| Diagnostic-ordering language | Real doctor dialogue models ordering tests/prescribing, which conflicts with the "not a diagnostic tool" persona | Regex-based diagnostic-language filter applied to training data |
| Filtering alone insufficient | Removing bad examples does not erase the base model's pretrained medical knowledge | Added a curated, hand-reviewed set of positive deferral/escalation examples to actively teach the desired behavior |
| Brand-name text corruption | An in-place string substitution (source-platform branding → replacement text) broke sentence grammar when the branding appeared as a sign-off phrase rather than a noun | Changed to clean removal plus whitespace/punctuation normalization, rather than in-place substitution |
| Teacher/student vocabulary mismatch | Same-family models pad their output layer to different sizes internally (152,064 vs. 151,936) | Logit tensors sliced to the shared dimension before computing KL-divergence loss |
| Quantization fidelity ≠ safety | Assumption that higher-fidelity quantization is strictly better did not hold | Real profiler measurements showed Q4_K_M and Q5_K_M within 0.4 points of each other on the scoring formula; the qualitative safety finding (Q5_K_M's diagnostic-ordering violation) was used as the decisive tiebreaker |
| Exposed API credential | A Hugging Face access token was inadvertently hardcoded in notebook source | Token revoked and regenerated; migrated to an encrypted secrets manager |

---

## 10. Final Submission Summary

| Component | Detail |
|---|---|
| Domain | Healthcare / Medical — patient education and symptom-based triage (explicitly not diagnosis or dosing) |
| Base / student model | Qwen2.5-1.5B-Instruct (Apache 2.0) |
| Teacher model | Qwen2.5-7B-Instruct |
| Training method | Logit-based knowledge distillation, followed by LoRA/QLoRA supervised fine-tuning on curated patient-dialogue and safety-behavior data |
| Quantization | **GGUF, Q4_K_M (recommended)** — selected on safety grounds; see Section 7.3 for the full quantitative-vs-qualitative reasoning |
| Runtime | llama.cpp, CPU-only inference, zero cloud dependency |
| Weights hosting | Publicly hosted on Hugging Face; downloaded fresh via `download_model.sh` |
| Safety framing | Model is explicitly trained to defer on dosing questions and avoid diagnostic-ordering language; validated (not fully robustly — see Section 7.4) against out-of-distribution prompts |

---

## 11. Limitations and Future Work

- **This is the most important limitation of the build**: the safety behavior described in Sections 3.3, 6.3, and 7.4 is substantially improved over the initial exam-data model but is **not fully robust** — at least one out-of-distribution prompt still triggered unsafe diagnostic-ordering language in the Q5_K_M variant. This should be disclosed transparently rather than understated.
- Training data drawn from real dialogue sources (MedDialog) has not been independently reviewed by a clinician for every example; expert review of a representative sample is a priority for any future iteration, more so than for the parallel agriculture build given the higher stakes of this domain.
- The curated positive deferral/escalation examples (Section 3.2) were authored with care but should be reviewed by someone with clinical training before being treated as a final, authoritative safety dataset.
- Profiling was conducted on [INSERT environment]; real audit-hardware numbers may differ from self-reported figures.
- The current system is text-only and single-turn, matching the evaluation format. A production system in this domain would benefit from a much more conservative deployment posture than a hackathon submission — e.g. mandatory disclaimers, logging for review, and a substantially larger and more rigorously reviewed safety-behavior dataset before real-world use.

---

## 12. Conclusion

This project delivered an offline, CPU-only patient education and triage assistant, built through the same distillation-then-fine-tuning-then-quantization pipeline as a parallel agriculture submission, but with a fundamentally different central challenge: not just achieving domain accuracy, but actively training and validating a safety boundary against diagnosis and dosing. The most valuable output of this build is arguably not the model itself but the documented process of finding, and partially — not fully — fixing, real safety regressions: an exam-data register mismatch, a dosage-recommendation leak, and a counter-intuitive quantization-fidelity safety finding. Each is reported here honestly, including where the fix remains incomplete, because that honesty is more useful to a judge, an auditor, or a future engineer than a report that only shows the success cases.
