# GEMINI.md — Typology Classification Agent (TCA)
# v3 Architecture — Updated April 27, 2026

---

## Project Overview

**Developer:** Emma Chen, Senior Data Scientist, KeyBank Financial Crimes Risk Management
**Environment:** Vertex AI Workbench (GCP), n2-standard-64, Gemini CLI (local Windows laptop)
**Stack:** Python, Google ADK, Gemini 2.5 Flash, BigQuery, Vertex AI text embeddings
**Repo:** fraud-uncertainty-ensembles, path: tca/
**Workbench path:** /home/jupyter/custom_agent/typology_agent/
**GCP Project:** taw-rmafincrime-prod-1398, us-central1
**BQ Project:** esa-src-prod-5736

**Purpose:** Build a typology knowledge base from historical SAR narratives that enables
classify() to score unseen narratives against learned AML typology indicator profiles,
supporting ASM missed-SAR analysis and candidate feature identification.

---

## Project Objective (Director-Confirmed)

**Primary application: ASM improvement**
- Simulate raising ASM auto-clear threshold (e.g. 30% → 50%)
- Retroactively identify cases that would have been auto-cleared but were actually filed as SARs (known misses)
- Run classify() on those missed cases to extract their indicator profiles
- Compare indicator profiles of missed SARs vs correctly cleared cases
- Indicators enriched in missed cases = candidate ASM features

**Secondary application: Investigations team**
- Within a typology, identify which indicators are dominant vs rare
- Support emergent threat detection via indicator vocabulary coverage analysis
- Meeting with investigations team pending

---

## Architecture: v3 (Current — Implemented)

### Core Design Principles
1. LLM describes behavior freely — never picks from vocabulary list
2. Python matches vocabulary — cosine similarity, never LLM judgment
3. All rule assembly, IDs, frequency counts = Python only
4. FinCEN tags = provenance only, never input filter
5. Thresholds = empirically derived, never expert-set
6. SME rules = post-hoc benchmark only, never system input
7. Phase 1 is append-only → Phase 2 consolidates once
8. No LLM sees the existing Rules Hypothesis during induction

### induct() — Two-Phase Pipeline

#### Phase 1: Per-Narrative Processing (in agent.py)

**Step 1: Build active_fincen_tags (Python)**
Extract FinCEN codes from BigQuery row. Provenance tracking only.
Filter out x999/x9999 catch-all codes.

**Step 2: parse_narrative() (Python — working)**
Split into sections: Introduction, Details of Investigation,
Conclusion, Contents of File.
Extract suspicious_total and review_period via regex.
NOTE: suspicious_total and review_period currently returning null
on most narratives — known issue, not yet fixed.

**Step 3: srl_extract() (Python — working)**
spaCy on Introduction + Conclusion only.
Extracts amounts, channels, transaction_types, red_flags
as lightweight supplement.

**Step 4: extract_behavioral_signals() — LLM Call 1 (BUILT)**
Location: tca/llm_helpers.py
Input: narrative_sections (introduction, conclusion, details_of_investigation)
Output: flat JSON list of free-form behavioral signal strings
Prompt: EXTRACT_BEHAVIORAL_SIGNALS_PROMPT
Rules: no proper nouns, no dollar amounts, no country names,
no case-specific details. No cap on signal count.
Returns: List[str]

**Step 5: match_signals_to_vocabulary() — Python cosine similarity (BUILT)**
Location: tca/sar_procedures.py
Input: behavioral_signals list from Step 4
Uses: Vertex AI text-embedding-005, SEMANTIC_SIMILARITY task type
Vocabulary: filtered_vocab (299 indicators after exclusions)
TOP_K = 3 matches per signal
SOFT_FLOOR = 0.65 minimum similarity score
Vocabulary embeddings cached to:
  tca/data/filtered_vocab_embeddings.npy
  tca/data/filtered_vocab_ids.npy
Cache validated by comparing stored IDs to current vocabulary IDs
on each module load — auto re-embeds if vocabulary changes.
Returns: List[Dict] with behavioral_signal, indicator_id, similarity_score
Sorted descending by similarity_score.

**Step 6: Append to raw_indicator_matches.json (Python)**
Location: tca/data/raw_indicator_matches.json
Schema:
{
  "case_id": "AML_INV_00000058210",
  "narrative_date": "",
  "active_fincen_tags": [
    {"fincen_tag": "821", "typology": "MONEY_LAUNDERING",
     "description": "(821)Suspicious concerning the source of funds"}
  ],
  "suspicious_total": null,
  "review_period": {"start": null, "end": null},
  "spacy_frames": {
    "amounts": [...], "channels": [...],
    "transaction_types": [...], "red_flags": [...]
  },
  "behavioral_signals": [
    "excessive P2P deposits from an apparently closed business entity",
    "large outgoing wire transaction funded by an unknown source of funds",
    ...
  ],
  "matched_indicators": [
    {"behavioral_signal": "...", "indicator_id": "KBP_3_k",
     "similarity_score": 0.828659},
    ...
  ]
}

#### Phase 2: consolidate_hypothesis() — Python only (BUILT)

Location: tca/agent.py — method on TypologyClassificationAgent
Called synchronously (no await) after induct() completes.
Input: complete raw_indicator_matches.json
Output: tca/data/rules_hypothesis.json

Process:
1. Build per-tag case population counts
   tag_case_populations[fincen_tag] = set of case_ids carrying that tag
2. For each record, find best similarity score per indicator per case
3. Fan out to every FinCEN tag the case carries (Option A)
   — each case contributes to all tag profiles it has tags for
4. Group by indicator_id within each tag profile
5. Compute per indicator per tag:
   - frequency = distinct case count
   - frequency_pct = frequency / total_cases_with_that_tag
   - avg_similarity_score = mean of best-match score per case
   - supporting_case_ids = list of case_ids
6. Sort by frequency desc, then avg_similarity_score desc
7. Write rules_hypothesis.json

Output schema:
{
  "821": {
    "total_cases": 10,
    "indicators": [
      {
        "indicator_id": "KBP_3_k",
        "indicator_text": "Fund transfers with no apparent economic...",
        "source": "KEYBANK_SAR_PROCEDURE",
        "frequency": 9,
        "frequency_pct": 0.9,
        "avg_similarity_score": 0.840286,
        "supporting_case_ids": ["AML_INV_...", ...]
      },
      ...
    ]
  },
  "925": { ... },
  ...
}

NOTE: With 10 narratives, 821 profile shows ~83 indicators.
frequency_pct is the primary signal quality filter —
no minimum frequency cutoff applied, frequency_pct handles separation
at scale.

---

## What's Working vs Pending

### WORKING ✅
- get_sar_narratives() — BigQuery fetch, SAR Monitoring Cases excluded
- get_fincen_tags() — target_fincen_tag filter working
- parse_narrative() — section splitting (suspicious_total/review_period null issue known)
- srl_extract() — spaCy extraction on Intro + Conclusion
- extract_behavioral_signals() — LLM call, clean generalizable signals
- match_signals_to_vocabulary() — TOP_K=3, SOFT_FLOOR=0.65, cache working
- Vocabulary embedding cache — .npy files at tca/data/
- consolidate_hypothesis() — rules_hypothesis.json building correctly
- induct_tester.py — runs induct() then consolidate_hypothesis() sequentially
- raw_indicator_matches.json — v3 schema with behavioral_signals field
- rules_hypothesis.json — per-tag indicator profiles with frequency stats
- ffiec_appendix_f.json — FFIEC Appendix F (verbatim, filtered)
- ffiec_appendix_g.json — FFIEC Appendix G (verbatim)
- sar_procedure.json — two-section structure (Section 9 + Section 10)

### IN PROGRESS / KNOWN ISSUES ⚙️
- suspicious_total null on most narratives — parse_narrative regex
  not capturing all narrative formats
- review_period null on most narratives — same issue
- Vocabulary gap: derogatory information / prior criminal history
  not covered by FFIEC F/G or SAR procedure vocabulary

### NOT YET BUILT ⏳
- classify() — score unseen narrative against rules_hypothesis.json
- consolidate_hypothesis() minimum frequency filter (deferred —
  frequency_pct handles at scale)
- Braintrust observability — stubbed with DummySpan
- GitLab access
- Investigations team application (meeting pending)
- Emergent threat / synthetic SAR eval (director suggestion)

---

## File Structure


tca/
├── agent.py                  # TypologyClassificationAgent
│                             # induct() — v3 pipeline ✅
│                             # consolidate_hypothesis() ✅
│                             # classify() — stub only
├── app_config.py             # Config loading
├── llm_helpers.py            # extract_behavioral_signals() ✅
│                             # match_indicators() — v2, deprecated
│                             # classify_narrative() — stub
├── parse_narrative.py        # ✅ Unchanged, working
├── srl_extract.py            # ✅ Unchanged, working
├── sar_procedures.py         # HIGH_RISK_COUNTRIES ✅
│                             # get_indicator_vocabulary() ✅
│                             # match_signals_to_vocabulary() ✅
│                             # get_filtered_vocab() ✅
│                             # Vocabulary embedding cache ✅
├── typology_keywords.py      # Unchanged
├── data/
│   ├── sar_procedure.json            # ✅ KeyBank procedures
│   ├── ffiec_appendix_f.json         # ✅ FFIEC Appendix F
│   ├── ffiec_appendix_g.json         # ✅ FFIEC Appendix G
│   ├── raw_indicator_matches.json    # ✅ Phase 1 output
│   ├── rules_hypothesis.json         # ✅ Phase 2 output
│   ├── filtered_vocab_embeddings.npy # ✅ Cached embeddings
│   └── filtered_vocab_ids.npy        # ✅ Cached vocab IDs
└── tools/
├── bq_tools.py
├── fincen_tag_tools.py           # ✅ get_fincen_tags() working
├── gcs_tools.py
└── sar_narrative_tools.py        # ✅ get_sar_narratives() working
Root level
induct_tester.py              # Runs induct() then consolidate_hypothesis()
narrative_inspector.py        # Fetches narrative + matched indicators by case_id
text_embedding_tester.ipynb   # Embedding/matching validation notebook
GEMINI.md                     # This file

---

## Embedding Setup

```python
import vertexai
from vertexai.language_models import TextEmbeddingInput, TextEmbeddingModel
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

vertexai.init(project="taw-rmafincrime-prod-1398", location="us-central1")
model = TextEmbeddingModel.from_pretrained("text-embedding-005")

def get_embeddings(texts, batch_size=20):
    all_embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        inputs = [TextEmbeddingInput(t, "SEMANTIC_SIMILARITY") for t in batch]
        embeddings = model.get_embeddings(inputs)
        all_embeddings.extend([e.values for e in embeddings])
    return np.array(all_embeddings)
```

Note: sentence-transformers installed but HuggingFace unreachable through proxy.
Use Vertex AI text-embedding-005 — better quality, no download needed.
Batch size increased from 5 to 20 for efficiency.

---

## Config (config.yaml)

```yaml
genai:
  model: gemini-2.5-flash
bigquery:
  bq_project_sar: esa-src-prod-5736
  bq_location: US
  bq_dataset_sar: ACT_SRC_DATA
defaults:
  default_start_date: "2026-01-01"
  default_end_date: "2026-03-31"
  target_fincen_tag: "821"
```

---

## Immediate Next Steps (Priority Order)

### 1. Build classify()
Apply rules_hypothesis.json indicator profiles to unseen narratives.
Pipeline:
- Run extract_behavioral_signals() on input narrative
- Run match_signals_to_vocabulary() on signals
- For each relevant FinCEN tag profile in rules_hypothesis:
  - Compute overlap between narrative's matched indicators
    and tag profile indicators
  - Weight by frequency_pct and avg_similarity_score
  - Produce per-tag similarity score
- Output: ranked tag profile matches with supporting indicators

### 2. Fix suspicious_total and review_period null issue
parse_narrative() regex not capturing all narrative formats.
Needs investigation across narrative samples to identify
format variants causing null returns.

### 3. Address derogatory information vocabulary gap
"Customer has derogatory information involving conspiracy to
commit money laundering" — no matching indicator in current vocabulary.
Options:
- Add supplementary indicator to sar_procedure.json
- Add custom indicator JSON file for gap categories

### 4. Scale induct() to larger corpus
Currently validated at limit=10.
Run at limit=50, then limit=100+ to improve frequency distribution
and separate signal from noise in rules_hypothesis.json.

### 5. Investigations team meeting
Discuss indicator dominance analysis and emergent threat detection
application. Pending scheduling.

---

## Key Design Decisions (Do Not Violate)

1. LLM describes behavior freely — never picks from vocabulary list
2. Python matches vocabulary — cosine similarity, never LLM judgment
3. All rule assembly, IDs, thresholds, frequency counts = Python only
4. FinCEN tags = provenance only, never input filter
5. Thresholds = empirically derived — min(suspicious_total) across
   supporting cases. Never expert-set. (deferred until null issue fixed)
6. SME rules = post-hoc benchmark only, never system input
7. Phase 1 is append-only → Phase 2 consolidates once
8. No LLM sees existing Rules Hypothesis during induction
9. TOP_K=3 per signal, SOFT_FLOOR=0.65 — tunable constants
10. frequency_pct is primary quality filter — no hard frequency cutoff
11. Each narrative fans out to ALL its FinCEN tags in consolidation
12. frequency counts distinct case_ids — not raw match rows

---

## FinCEN Code Map (Key Codes for 821 Corpus)

- 821 = Suspicious concerning the source of funds
- 824 = Funnel account
- 805 = Suspicious EFT/wire transfers
- 812 = Transaction out of pattern for customer(s)
- 807 = Suspicious use of multiple accounts
- 925 = Transaction with no apparent economic, business, or lawful purpose
- 911 = Two or more individuals working together
- 928 = Transaction(s) involving foreign high risk jurisdiction
- x999/x9999 = Other catch-alls (filtered out)

---

## High-Risk Country List (KeyBank Official)

Afghanistan, Belarus, Central African Republic, Congo (DRC),
Cuba, Ethiopia, Iran, Iraq, Libya, Mali, Myanmar (Burma),
Nicaragua, North Korea, Russia, Somalia, Sudan, South Sudan,
Syria, Ukraine (Crimea/DNR/LNR), Venezuela, Yemen

Note: Balkans excluded. China and Mexico NOT on list.
In all outputs: HRC country name → "high-risk foreign jurisdiction"

---

## Gemini CLI Notes

- Running on Windows laptop (local)
- Workspace pointed at typology_agent directory
- Model: Auto (Gemini 2.5)
- GEMINI.md is the context file Gemini CLI reads on every session
- Always read GEMINI.md before any coding task
- Gemini CLI edits files locally — copy to Workbench to run
- Workbench runs code — Gemini CLI has no network access to BQ or Vertex AI
- Working directory on Workbench: /home/jupyter/custom_agent
- os.chdir("/home/jupyter/custom_agent/typology_agent") at top of
  any notebook to ensure consistent relative paths

