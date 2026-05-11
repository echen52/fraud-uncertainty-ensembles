# GEMINI.md
# v4 Architecture — Updated May 11, 2026

---

## Project Overview

- **Environment:** Vertex AI Workbench (GCP), n2-standard-64, Gemini CLI (local Windows laptop)
- **Stack:** Python, Google ADK, Gemini 2.5 Flash, BigQuery, Vertex AI text-embedding-005
- **Repo:** fraud-uncertainty-ensembles, path: tca/
- **Workbench path:** /home/jupyter/custom_agent/typology_agent/
- **GCP Project:** taw-rmafincrime-prod-1398, us-central1
- **BQ Project:** esa-src-prod-5736, dataset: ACT_SRC_DATA
- **Gemini CLI:** runs locally on Windows; workspace = typology_agent dir; model = Auto (Gemini 2.5)
- **Working directory note:** os.chdir("/home/jupyter/custom_agent/typology_agent") required at top of all notebooks

**Purpose:** Build a typology knowledge base from historical SAR narratives that enables
`classify()` to score unseen narratives against learned AML typology indicator profiles,
supporting ASM missed-SAR analysis and candidate feature identification.

A second use case is a well-reading agent that extracts structured qualitative details
from SAR narratives to support the investigations team as they develop tracking procedures.
Currently they manually fill in dashboards with details as they read narratives one at a time.

A third use case is a separate Reasoning Agent process (do not conflate with TCA methods)
aimed at three-tier case disposition: Auto SAR, Auto Clear, and SME Review Required.
This is under active design and is built separately from induct()/classify().

---

## Project Use Cases (Director-Confirmed as of 2026-05-11)

Three confirmed use cases. TCA (induct/classify) directly supports Use Cases 1 and 2.
Use Case 3 is a separate agent process — do NOT modify induct()/classify() for it.

### Use Case 1 — ASM Distribution by Typology Code
Handed off to Lei's team. Not an active TCA work item.
Granular typology codes produced by TCA's FinCEN remapping work are the input.
Lei's team examines ASM score distribution per typology code to identify segments
where ASM underperforms. How to improve ASM features is determined after that analysis.

### Use Case 2 — ETSO Trend Analysis (Active TCA Work Item)
ETSO team has provided their existing NON TM SAR workbook — a manually maintained
trend analysis covering special case SAR narratives. Fields currently tracked include:
Law Enforcement referrals, case type, first-time SAR flag, total SAR amount, channels,
and others. Many fields are currently manually input.

Task: extend induct() and related agent methods to pull additional data from BQ,
fill gaps in the workbook, and automate what is currently manually input.
This builds directly on the existing FinCEN 'Other' remapping work
(fincen_distribution_analyzer_v3.py).
Deliverable: a more streamlined, BQ-driven version of the ETSO workbook,
shared as a proof of concept to identify enhancements.

### Use Case 3 — Reasoning Agent (Separate Process from TCA)
A new agent process to be built separately from TCA. Do not modify induct()/classify()
for this use case — keep TCA methods clean and isolated.

Goal: three-tier case disposition system:
  - Auto SAR
  - Auto Clear
  - SME Review Required

Hypothesis: variety and count of indicators across multiple typologies correlates
with SAR likelihood. Needs quantification. This is the overarching design goal
for the Reasoning Agent.

#### Sub-case A: Qualitative Typologies
Examples: Law Enforcement (9003), Marijuana, Drug Trafficking.

Narratives for these typologies focus on investigative process and behavioral context
rather than transaction values. Qualitative indicators are well-captured in SAR
narratives via induct()-style reading.

Known inference-time flaw: at point of new case review, no narrative exists —
qualitative indicators cannot be searched or applied directly to a cold case.

Director's reframe: use narrative reading across historical SARs for these typologies
to extract and solidify the investigative PROCESS — what do investigators consistently
look for and confirm? Output is a documented process pattern per typology, not a
runtime scoring signal. This process pattern becomes the grounding for the Reasoning
Agent when reviewing new cases of that typology.

Example for Marijuana: if all marijuana SARs involve a negative news check, that
confirms the process. If some do not, we are picking up additional information about
variation in the filing process for that typology. The agent reads narratives as
training examples to confirm or surface the process.

#### Sub-case B: Quantitative Typologies
Examples: 824 Funnel Account (warm start), future cold-start typologies.

Warm start: existing alerting rule logic is available as grounding for the agent.
Cold start: no existing SME rule — agent must induce rules from narratives only.
821 (Suspicious source of funds, 7.7K SARs) is too broadly defined and too high
volume for narrative induction to produce reliable rules — defer.

Immediate pilot: 824 Funnel Account (warm start).
  - Alerting rule parameters (velocity, amount, counterparty thresholds) —
    to be provided by investigations team. PENDING.
  - Non-SAR filed cases for 824 requested from David Rowland. PENDING.
  - Approach: compare SAR-filed vs. non-SAR-filed cases that triggered the
    same funnel account alert. Use narrative reading to identify what
    differentiates alert → SAR from alert → no SAR.
  - This is the primary mechanism for understanding how to move from alerting
    to SAR disposition using narrative-derived signal.

#### Iterative Narrative Reading — Design Decision
Early discussion raised the idea of reading narrative 1 then narrative 2 within
the same typology to accumulate a typology profile iteratively (sequential).

DECISION: Do not use iterative/sequential design. Iterative design was attempted
in TCA Phase 1 and did not perform well. Use batch reading approach — read all
narratives for a typology in one consolidation pass, consistent with how
consolidate_hypothesis() operates in TCA. Flag for director alignment if needed.

---

## Project Objective (Director-Confirmed)

**Primary application: ASM Improvement**
- Simulate raising ASM auto-clear threshold (e.g. 30% → 50%)
- Retroactively identify cases that would have been auto-cleared but were
  actually filed as SARs (known misses)
- Run classify() on those missed cases to extract their indicator profiles
- Compare indicator profiles of missed SARs vs correctly cleared cases
- Indicators enriched in missed cases = candidate ASM features

**Secondary application: Investigations team extraction**
- Within a typology, identify which indicators are dominant vs rare
- Extract structured qualitative details from SAR narratives to replace
  manual dashboard filling for the investigations team
- Support emergent threat detection via indicator vocabulary coverage analysis

---

## Architecture: v3 (Current — Implemented)

### Core Design Principles (Do Not Violate)
1. LLM describes behavior freely — never picks from vocabulary list
2. Python matches vocabulary via cosine similarity, never LLM judgment
3. All rule assembly, IDs, frequency counts = Python only
4. FinCEN tags = provenance only, never input filter
5. Thresholds = empirically derived, never expert-set
6. SME rules = post-hoc benchmark only, never system input
7. induct() Phase 1 is append-only — consolidate_hypothesis() consolidates once
8. No LLM sees the existing Rules Hypothesis during induction
9. frequency_pct is the primary quality filter — no hard frequency cutoff
10. Frequency counts distinct case_ids, not raw match rows

---

### `induct()` — Two-Phase Pipeline

#### Phase 1: Per-Narrative Processing (agent.py)

**Step 1: build_active_fincen_tags() — Python**
Extract FinCEN codes from BigQuery row. Provenance tracking only.
Generic '99' codes (3999, 9999, etc.) remapped to synthetic descriptive codes
via regex on _OTH free-text fields. Applied pre-aggregation at case level.
Multi-code assignment via UNNEST(ARRAY(...)) in BQ.
Canonical implementation: fincen_distribution_analyzer_v3.py
Validated via: regex_traceability_reporter.py, fraud_regex_traceability_reporter.py

**Step 2: parse_narrative() — Python (KNOWN BUG)**
Split narrative into sections: Introduction, Details of Investigation,
Conclusion, Contents of File.
Extract suspicious_total and review_period via regex.
BUG: suspicious_total and review_period returning null on most narratives.
Regex not capturing all narrative formats. Active work item — not yet fixed.

**Step 3: srl_extract() — Python (spaCy)**
spaCy NER + noun chunk analysis on Introduction + Conclusion sections.
Extracts: amounts, dates, channels, transaction types, red flags.
spaCy 3.8.14 + en_core_web_sm installed.
torch, transformers, allennlp NOT available (proxy/size constraints).

**Step 4: extract_behavioral_signals() — LLM call (Gemini)**
Gemini generates free-text behavioral signal descriptions.
No PII, no amounts, no countries in output.
LLM describes behavior only — never selects from vocabulary.
Few-shot examples grounded across typology-diverse narratives.

**Step 5: match_signals_to_vocabulary() — Python (cosine similarity)**
Matches LLM-generated descriptions to indicator vocabulary.
TOP_K = 3, SOFT_FLOOR = 0.65
Embedding cache stored at tca/data/ (.npy files)
Model: Vertex AI text-embedding-005

**Step 6: Append to raw_indicator_matches.json**
v3 schema includes behavioral_signals field.
Output location: tca/output/raw_indicator_matches.json
Each narrative fans out to ALL its FinCEN tags (multi-tag fan-out).

#### Phase 2: consolidate_hypothesis() — Python only
Runs synchronously after induct() completes (via induct_tester.py).
Input: raw_indicator_matches.json
Output: rules_hypothesis.json (tca/output/)
Per-FinCEN-tag indicator profiles with:
  - frequency (count of distinct case_ids)
  - frequency_pct (primary quality filter)
  - avg_similarity_score
  - supporting_case_ids
No minimum frequency cutoff — frequency_pct handles separation at scale.
Indicator restriction logic for multi-tag narratives: OPEN WORK ITEM.
Risk: narratives carrying many FinCEN tags dilute indicator profiles
across all tags they touch (overclassification). Resolution pending.

---

### `classify()` — Unseen Narrative Scoring
Status: working, successfully classifying narratives.
NOT in active development focus unless issues arise with induct() or
consolidate_hypothesis(), or the new investigations extraction function.

Rule matching as primary signal; embedding similarity as backup.
LLM calls limited to confirmation/rejection of programmatically-identified candidates.
Returns structured indicator match profile per case (not a raw probability score).

---

## Reference Data Files

- ffiec_appendix_f.json — FFIEC Appendix F (verbatim, filtered), tca/data/
- ffiec_appendix_g.json — FFIEC Appendix G (verbatim), tca/data/
- sar_procedure.json — KeyBank SAR procedure, Section 9 + Section 10, tca/data/

---

## FinCEN 'Other' Tag Remapping

Generic '99' codes (e.g. 3999, 9999) remapped to synthetic descriptive codes
via regex on _OTH free-text fields. Applied pre-aggregation at case level.
Multi-code assignment via UNNEST(ARRAY(...)) in BQ ("combo inflation").
Canonical implementation: fincen_distribution_analyzer_v3.py
Validated via: regex_traceability_reporter.py, fraud_regex_traceability_reporter.py

---

## High-Risk Country List (KeyBank Official) — PENDING UPDATE

Current list: Afghanistan, Belarus, CAR, DRC, Cuba, Ethiopia, Iran, Iraq,
Libya, Mali, Myanmar, Nicaragua, North Korea, Russia, Somalia, Sudan,
South Sudan, Syria, Ukraine (Crimea/DNR/LNR), Venezuela, Yemen.
Balkans excluded. China and Mexico NOT on list.
In all outputs: HRC country name → "high-risk foreign jurisdiction"
STATUS: Update to this list is pending. Do not treat as final.

---

## What's Working vs Pending

### WORKING ✅
- get_sar_narratives() — BigQuery fetch, SAR Monitoring Cases excluded
- get_fincen_tags() — target_fincen_tag filter working
- parse_narrative() — section splitting working (suspicious_total/review_period null bug known)
- srl_extract() — spaCy extraction on Intro + Conclusion
- extract_behavioral_signals() — LLM call, clean generalizable signals
- match_signals_to_vocabulary() — TOP_K=3, SOFT_FLOOR=0.65, cache working
- Vocabulary embedding cache — .npy files at tca/data/
- consolidate_hypothesis() — rules_hypothesis.json building correctly
- induct_tester.py — runs induct() then consolidate_hypothesis() sequentially
- raw_indicator_matches.json — v3 schema with behavioral_signals field
- rules_hypothesis.json — per-tag indicator profiles with frequency stats
- classify() — working, not in active scope
- Induction results — 1000 narratives processed for 2025
- ffiec_appendix_f.json, ffiec_appendix_g.json, sar_procedure.json
- FinCEN 'Other' tag remapping — fincen_distribution_analyzer_v3.py validated

### ACTIVE WORK ITEMS ⚙️
- Fix suspicious_total / review_period null bug in parse_narrative() —
  regex not capturing all narrative formats
- Build investigations team extraction function in/alongside induct() —
  filed-on instrument, repeat/first-time filing, BQ merge features,
  structured qualitative details (Use Case 2 / ETSO workbook automation)
- Indicator restriction logic in consolidate_hypothesis() —
  prevent overclassification when narratives carry multiple FinCEN tags

### PENDING EXTERNAL DEPENDENCIES ⏳
- 824 Funnel Account alerting rule parameters — from investigations team
- Non-SAR filed cases for 824 — from David Rowland
- High-Risk Country List update — pending
- ETSO NON TM SAR workbook field mapping — confirm all fields to automate

---

## Open Design Questions (as of 2026-05-11)

- Iterative vs. batch narrative reading for Reasoning Agent (Use Case 3) —
  iterative design did not perform well in TCA Phase 1; batch preferred.
  Confirm with director before committing to design.
- Quantification of multi-typology indicator accumulation for three-tier
  disposition — how to score variety and count of indicators across typologies
  into Auto SAR / Auto Clear / SME thresholds. Not yet designed.
- Qualitative typology process extraction output format — what does a
  "documented process pattern" look like as a machine-usable artifact?
  Needs definition before building Sub-case A of Reasoning Agent.
- 824 warm-start grounding — confirm whether alerting rule logic should be
  injected into prompt as grounding, or used only as a post-hoc benchmark.
