# AIDOL — Health Advisor (Exploratory)

![Status](https://img.shields.io/badge/Status-Not%20Submitted%20to%20ADTC-critical)
![Domain](https://img.shields.io/badge/Domain-Healthcare%20%26%20Medical-red)
![Base Model](https://img.shields.io/badge/Base%20Model-Qwen2.5--1.5B--Instruct-8A2BE2)
![Teacher Model](https://img.shields.io/badge/Distillation%20Teacher-Qwen2.5--7B--Instruct-9370DB)
![Format](https://img.shields.io/badge/Format-GGUF-orange)
![Runtime](https://img.shields.io/badge/Runtime-llama.cpp-000000)
![Quantization](https://img.shields.io/badge/Quantization-Q5__K__M-1E7A72)
![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?logo=pytorch&logoColor=white)
![Hugging Face](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-EYEDOL-FFD21E)
![License](https://img.shields.io/badge/License-Apache%202.0-green)

> ### ⚠️ This repository is NOT the team's ADTC 2026 competition submission.
> The team's official Gate 1 entry is the **Agriculture** domain model — see [`aidol-agri-advisor`](https://github.com/EYEDOL/aidol-agri-advisor). This repository documents exploratory healthcare-domain work, published for transparency. **Do not use this model for real medical guidance.** Read [Known Limitations](#known-limitations) below before going further.

**Team:** AIDOL &nbsp;·&nbsp; **Submitter:** Adegoke Israel Adedolapo &nbsp;·&nbsp; **Domain explored:** Healthcare & Medical (patient education / triage support scope)

---

## Architecture

![Healthcare exploration pipeline diagram](images/pipeline_healthcare.svg)

This model was built through the same distillation + fine-tuning + quantization pipeline as the team's submitted Agriculture model, developed in parallel specifically to compare domains before choosing a final Gate 1 entry.

---

## Known Limitations

Direct generation testing revealed two distinct, unresolved failure modes:

1. **Degenerate repetition (Q4_K_M).** On a pediatric-dehydration prompt, the model produced a repeating loop of the same sentence rather than a bounded, coherent answer.
2. **Diagnostic-ordering behavior (Q5_K_M, this repo).** On a red-flag-symptom prompt, the model recommended a specific invasive procedure ("get a lumbar puncture done"), directly contradicting its own system instruction that it is not a diagnostic tool.

Both failures persisted despite targeted training-data filtering (explicit dosage removal, diagnostic-ordering language removal). Neither failure was observed in the equivalently-trained Agriculture model, which is why Agriculture was selected as the team's actual submission.

**The automated accuracy benchmark alone does not catch this.** Both healthcare quantization variants scored *higher* (0.80) than the submitted Agriculture model (0.70) on the local `arc_easy` smoke test — a generic reasoning benchmark, not a domain-specific safety check. This repository is published specifically to document that gap honestly, not to present a benchmark score without its necessary context.

---

## Repository Structure

```
.
├── metadata.json          # Domain: healthcare_medical — explicitly marked non-final in cross_disciplinary_pairing
├── download_model.sh      # Downloads the quantized GGUF model — no credentials required
├── REPORT.md              # Full technical report, including detailed limitations
├── images/
│   └── pipeline_healthcare.svg
├── model/                 # Populated by download_model.sh — not committed to git
└── .gitignore
```

---

## Quick Start (for review/testing purposes only)

```bash
bash download_model.sh
adtc-profiler run --submission . --mode participant --output submission.json --skip-accuracy
```

---

## Benchmarks (measured)

| Metric | Q5_K_M (this repo) | Q4_K_M |
|---|---|---|
| Tokens/sec | 9.27 | 9.71 |
| Peak RAM | 1.23 GB | 1.67 GB |
| Accuracy (arc_easy smoke test) | 0.80 | 0.80 |
| Real generation quality | Unresolved diagnostic-ordering failure | Unresolved repetition-loop failure |

Full details, including what would be needed to make this submission-ready, in [`REPORT.md`](REPORT.md).

---

## License

Apache 2.0, consistent with the base model license.
