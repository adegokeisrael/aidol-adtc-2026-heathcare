# Technical Report — AIDOL Health Advisor (Exploratory)

**Team ID:** AIDOL
**Submitter:** Adegoke Israel Adedolapo
**Domain:** healthcare_medical
**Model:** AIDOL-Health-Advisor-1.5B-Q5_K_M

> **This is NOT the team's ADTC 2026 competition submission.** The team's official Gate 1 entry is the Agriculture domain model (see `aidol-agri-advisor`). This repository documents exploratory work developed alongside that submission, and is published for transparency. **Do not use this model for real medical guidance.** See "Known Limitations" below before going further.

---

## Problem

The same underlying goal as the agriculture work — an offline, single-turn advisory assistant for low-connectivity contexts — applied to patient education and triage support: helping a caregiver or patient understand symptoms, recognize red flags, and know when to seek in-person care, without cloud dependency. This is one of the domain's own listed sub-areas ("clinical information, medical Q&A, triage support, and patient education"), deliberately scoped away from diagnosis given the higher liability and safety bar clinical content carries.

---

## Design Decisions

- **Base model:** Qwen2.5-1.5B-Instruct, same rationale as the agriculture model (Apache 2.0, strong instruction-following per size, mature GGUF conversion path).
- **Training pipeline:** identical structure to the agriculture model — knowledge distillation from Qwen2.5-7B-Instruct (frozen 4-bit teacher, response-masked CE+KL loss), followed by domain SFT (QLoRA, rank 16).
- **Datasets:** initial attempts used MedMCQA and MedQA-USMLE (physician licensing exam question banks), which produced a serious register mismatch — see Known Limitations. The dataset was subsequently swapped to `khoaliamle/MedDialog-EN-100k` (real doctor-patient chat consultations), with two cleaning passes: (1) explicit numeric drug dosages filtered out of training data, and (2) diagnostic-ordering language (e.g. "get a CT scan," "I advise you to get a blood test") filtered out, to discourage the model from imitating the ordering/prescribing behavior real doctor responses naturally contain.
- **System persona:** explicitly instructed as "NOT a diagnostic tool," directed to explain, flag red-flag symptoms, and defer specific dosing questions to a clinician or pharmacist.

---

## Known Limitations — Why This Was Not Submitted

Direct generation testing against held-out prompts revealed two distinct, unresolved failure modes, present even after the data-cleaning passes described above:

1. **Degenerate repetition (Q4_K_M variant).** On a prompt about pediatric dehydration signs, the model entered a repetition loop, producing the same sentence fragment multiple times in sequence rather than a coherent, bounded answer.
2. **Diagnostic-ordering behavior surviving filtering (Q5_K_M variant, the model in this repository).** On a prompt asking for red-flag symptoms requiring urgent referral, the model responded by recommending a specific invasive diagnostic procedure ("I would advise you to get a lumbar puncture done to rule out meningitis") — directly contradicting its own system instruction not to act as a diagnostic tool. This is assessed as a genuine, unresolved safety-relevant failure, not a minor style issue.

Neither failure mode was observed in the equivalently-trained Agriculture model, which is why Agriculture was selected as the team's actual competition submission. Further work (a larger and more rigorously reviewed dataset, more aggressive filtering, and independent clinical review of training content) would be needed before this model should be considered for any real use.

---

## Constraints

Same hardware/runtime constraints as the agriculture submission: ADTC Standard Laptop target (8GB DDR4, integrated graphics, Ubuntu 22.04), llama.cpp/GGUF only, English-only evaluation, 2,048-token context ceiling. Development and profiling were carried out on a cloud CPU environment (Kaggle) as a proxy for the target hardware.

---

## Benchmarks

Measured via `adtc-profiler run --mode participant`, full run including accuracy.

| Metric | Q5_K_M (this repo) | Q4_K_M (for comparison) |
|---|---|---|
| Generation speed | 9.27 tokens/sec | 9.71 tokens/sec |
| Peak RSS | 1257.85 MB (≈1.23 GB) | 1714.34 MB (≈1.67 GB) |
| Accuracy (arc_easy, 50-sample smoke test) | 0.80 | 0.80 |
| CPU utilization (p99) | 59.9% | 60.3% |
| Thermal throttling | Not detected (core temp reads `null` — cloud CPU proxy, no exposed hardware sensors) | Not detected (same caveat) |

**Estimated total score** (official formula, both variants): Q5_K_M ≈ 75.0, Q4_K_M ≈ 74.6 — both numerically higher than the submitted Agriculture model's ≈70.7. **This is precisely why the automated benchmark score alone should not be trusted as a substitute for direct qualitative review** — it reflects a generic 50-sample reasoning smoke test, not the domain-specific, safety-relevant behavior documented above, which does not appear in this automated number at all.

---

## What Would Be Needed to Make This Submission-Ready

1. Independent clinical review of a representative training-data sample against a formal reference (e.g. WHO guidelines).
2. Stronger, more targeted filtering or additional deferral-behavior training examples specifically addressing diagnostic-ordering language.
3. A larger held-out qualitative test set to confirm the repetition-loop failure mode is fully resolved, not just reduced in frequency.
4. Re-evaluation against fresh, out-of-distribution prompts before any consideration of real-world or competition use.
