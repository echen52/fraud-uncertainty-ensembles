# TCA (Typology Classification Agent) — Design History
## induct() Architecture Evolution: v1 → v2 → v3

---

## v1 — LLM-Driven Rule Synthesis
**Status:** Abandoned after limit=10 evaluation  
**Period:** ~April 7–17, 2026

### Architecture Overview
Two LLM calls per FinCEN tag per narrative. LLM responsible for all rule assembly, merging, schema construction, and hypothesis management.

### Pipeline Steps

**Step 1: Build active_fincen_tags (Python)**
Extract FinCEN tag codes and descriptions from BigQuery row. Used as input filter for LLM calls — each tag scoped the LLM's extraction separately.

**Step 2: parse_narrative() (Python)**
Split narrative into sections. Extract `suspicious_total` and `review_period` via regex.

**Step 3: srl_extract() (Python)**
Run spaCy on Introduction + Conclusion. Extract amounts, channels, transaction types, red flags.

**Step 4: get_procedures_for_tags() (Python)**
Load KeyBank SAR procedure text relevant to active FinCEN tags from `sar_procedure.json`. Injected as grounding into LLM calls.

**Step 5: Call 1 — extract_narrative_structure() (LLM)**
System prompt: EXTRACT_NARRATIVE_PROMPT
Input: Full narrative sections + spaCy frames + SAR procedures
Output: Structured JSON with extracted fields (amounts, channels, red flags, counterparties, typology signals)

**Step 6: Call 2 — synthesize_rules() (LLM)**
System prompt: SYNTHESIZE_RULES_PROMPT
Input: Extracted structure from Call 1 + existing Rules Hypothesis + SAR procedures + active_fincen_tags
Output: Updated Rules Hypothesis with new/merged rules
Called once per narrative with ALL active_fincen_tags passed together.

**Step 7: Update Rules Hypothesis in-memory (LLM-driven)**
LLM responsible for: rule ID generation, channel assignment, threshold setting, indicator deduplication, merging new rules into existing hypothesis.

### Output
`rules_hypothesis.json` — built incrementally, narrative by narrative.

### Discoveries and Problems Identified

**Problems:**
- **Cross-tag contamination:** Calling synthesize_rules() once per narrative with all tags together caused indicators from one FinCEN tag to bleed into rules for a different tag (e.g., 821 indicators appearing in 805 rules)
- **Indicator bloat:** LLM accumulated indicators rather than consolidating — 20-35 indicators per rule with supporting_case_count of 4-6
- **Case-specific entity leakage:** Named entities (business names, specific countries, platform names) appeared in indicators despite prompt instructions (e.g., "EXOTIC S SHOP INC", "PLAYERMATRIX", "China")
- **Inconsistent thresholds:** LLM set threshold from individual case amounts rather than computing minimum across cases — produced arbitrary values like $126.52, $4573.31
- **Schema violations:** Non-canonical channel values appearing (e.g., "Official Bank Check" instead of "Check")
- **Duplicate rule_ids:** LLM generated duplicate IDs across different FinCEN tags
- **OR logic contamination:** LLM combined multiple channels into one rule using OR logic
- **Mid-stream merge instability:** Passing existing rules hypothesis back to LLM each iteration caused drift and inconsistency

**Fixes attempted before abandoning:**
- Restructured to one synthesize_rules() call per tag per narrative (reduced cross-tag contamination but did not eliminate it)
- Added HIGH_RISK_COUNTRIES constant and HRC normalization instructions
- Added indicator cap instructions
- Added entity stripping instructions
Each fix introduced new problems — recognized as fundamental architectural flaw, not a prompt engineering problem.

**Root cause diagnosis:**
The LLM was being asked to do Python's job — rule assembly, merging, schema management, threshold computation, ID generation. These are deterministic operations that LLMs perform unreliably.

---

## v2 — Controlled Vocabulary Matching (Unflipped)
**Status:** Built and evaluated at limit=10. Partially working but insufficient for BOT package evaluation.
**Period:** ~April 18–23, 2026

### Key Architectural Changes from v1
- LLM scope reduced to single matching call per narrative (not per tag)
- FinCEN tags moved from input filter to output validation only
- All rule assembly moved to Python
- Rules Hypothesis built in single post-corpus consolidation pass (Phase 2), not incrementally
- Controlled indicator vocabulary introduced (FFIEC Appendix F + G + KeyBank SAR procedures)
- Two-phase induct() structure: Phase 1 (per-narrative, LLM involved) + Phase 2 (post-corpus, Python only)

### New Components Introduced
- `ffiec_appendix_f.json` — FFIEC BSA/AML Manual Appendix F red flags (75+ indicators, 15 categories, verbatim text, no typology_relevance pre-judgment)
- `ffiec_appendix_g.json` — FFIEC BSA/AML Manual Appendix G structuring definitions
- `get_indicator_vocabulary()` — combined flat vocabulary from all three sources
- `match_indicators()` in llm_helpers.py — single LLM call replacing both v1 calls
- `raw_indicator_matches.json` — intermediate append-only artifact from Phase 1
- `consolidate_hypothesis()` — Phase 2 Python-only consolidation (designed but not fully implemented before v3 decision)
- HIGH_RISK_COUNTRIES constant — KeyBank official list from SharePoint (Afghanistan, Belarus, CAR, Congo DRC, Cuba, Ethiopia, Iran, Iraq, Libya, Mali, Myanmar, Nicaragua, North Korea, Russia, Somalia, Sudan, South Sudan, Syria, Ukraine Crimea/DNR/LNR, Venezuela, Yemen)
- `narrative_inspector.py` — diagnostic tool for manual verification of evidence quotes against source narrative
- Vocabulary category filtering — excluded EFC, EMP, CBF, BLK, INS, LA, TRD as irrelevant to retail banking. Reduced vocabulary from 327 to ~150 indicators.

### Pipeline Steps

**Step 1: Build active_fincen_tags (Python)**
Same as v1 but now used for provenance tracking only — NOT used to scope LLM extraction.

**Step 2: parse_narrative() (Python)**
Unchanged from v1.

**Step 3: srl_extract() (Python)**
Unchanged from v1. Recognized as insufficient for quantitative pattern extraction but kept as lightweight supplement.

**Step 4: get_indicator_vocabulary() (Python)**
Load and merge all indicators from sar_procedure.json, ffiec_appendix_f.json, ffiec_appendix_g.json into a single flat list with indicator_id, text, source fields.

**Step 5: match_indicators() — Single LLM Call**
System prompt: MATCH_INDICATORS_PROMPT
Input: Introduction + Conclusion (primary), Details of Investigation (secondary), spaCy frames, full filtered vocabulary as numbered list
Output: Flat list of {indicator_id, evidence_quote} objects
LLM instructed to: match only from vocabulary, return JSON only, one sentence max per quote, no proper nouns, HRC countries replaced with "high-risk foreign jurisdiction"

**Step 6: Append to raw_indicator_matches.json (Python)**
Record structure:
```json
{
  "case_id": "...",
  "narrative_date": "...",
  "active_fincen_tags": [...],
  "suspicious_total": null,
  "review_period": {"start": "...", "end": "..."},
  "spacy_frames": {"amounts": [], "channels": [], "transaction_types": [], "red_flags": []},
  "matched_indicators": [{"indicator_id": "...", "evidence_quote": "..."}]
}
```

### consolidate_hypothesis() Design — Phase 2 (designed, not finalized before v3)
- Group matched_indicators by indicator_id across all records
- Count frequency per indicator
- Derive channel from spaCy frames (dominant channel across supporting cases)
- Compute threshold as min(suspicious_total) across supporting cases
- Generate rule_ids in Python
- Build Rules Hypothesis JSON

### Discoveries and Problems Identified

**Improvements over v1:**
- No cross-tag contamination
- No mid-stream merge instability
- No LLM-generated rule IDs, channels, or thresholds
- Indicator IDs traceable to authoritative sources (FFIEC, KeyBank)
- Evidence quotes verifiable against source narrative via narrative_inspector.py
- Matching quality good for rich narratives with specific language

**Remaining problems:**
- **Over-matching:** LLM matched thematically related indicators rather than specifically evidenced ones. FFI_F_SCA_007 (foreign correspondent bank) matched in domestic self-to-self P2P case. FFI_F_OUA_019 (account closure/dormancy) matched in active account case.
- **Evidence quote reuse:** Same quote used to justify 3-5 different indicators
- **Hallucinated indicator IDs:** KBP_3_k returned by LLM but not found in sar_procedure.json
- **suspicious_total null on most cases:** parse_narrative regex not reliably capturing total from all narrative formats
- **spaCy amount noise:** Dates captured as amounts ("4/28/2025 $"), transaction rows bleeding into transaction_types
- **Fundamental limitation — no quantitative extraction:** All matched_indicators qualitative only. Without transaction-level quantitative data, consolidate_hypothesis() cannot produce BOT-matchable rule conditions. Rules Hypothesis would only be qualitative typology documentation — insufficient for Agent 2 BOT package evaluation.

**Root cause of fundamental limitation:**
Details of Investigation transaction table — where quantitative behavioral patterns live — was not being read by the LLM. Introduction and Conclusion give qualitative summary signals only. The transaction table gives channel, direction, amount, counterparty type, velocity — the actual features Agent 2 needs to evaluate a BOT package.

**Critical design insight leading to v3:**
The matching approach (give LLM a vocabulary list, ask it to pick matches) is architecturally flawed:
1. LLM has too many options → weak thematic associations → over-matching
2. Matching quality degrades for simple/generic narratives lacking specific vocabulary-matching language

Alternative identified: flip the process — let LLM describe behavioral patterns in free-form language, then match against vocabulary using Python cosine similarity. This separates semantic understanding (LLM strength) from precise matching (Python strength).

---

## v3 — Flipped Architecture with Transaction Extraction
**Status:** Designed. Implementation pending.
**Period:** April 24, 2026

### Key Architectural Changes from v2
- Added dedicated transaction extraction LLM call (new Call 1) reading Details of Investigation transaction table and surrounding prose
- Replaced vocabulary matching LLM call with free-form behavioral signal extraction (new Call 2)
- Replaced LLM vocabulary matching with Python cosine similarity matching (new Step 6)
- Both quantitative (transaction patterns) and qualitative (behavioral signals) now captured per narrative
- consolidate_hypothesis() now has sufficient data to produce BOT-matchable rule conditions
- SME rules explicitly excluded from system inputs — reserved for post-hoc validation benchmark only

### Motivating Principles
- LLMs do what they are best at: reading format-inconsistent text, semantic understanding, free-form description
- Python does what it is best at: deterministic matching, statistics, structure, reproducibility
- No LLM is asked to pick from a large list — eliminates over-matching by design
- No LLM sets thresholds or generates IDs — eliminates arbitrary values
- Transaction-level quantitative patterns enable Agent 2 BOT package evaluation

### New Components
- `extract_transactions()` in llm_helpers.py — new LLM call for transaction table extraction
- `extract_behavioral_signals()` in llm_helpers.py — new LLM call replacing match_indicators()
- `match_vocabulary()` in agent.py or sar_procedures.py — Python cosine similarity matching
- Sentence embedding model for similarity computation (spaCy or lightweight transformer)
- Extended raw_indicator_matches.json schema with transaction_extraction and behavioral_signals fields

### Pipeline Steps

**Step 1: Build active_fincen_tags (Python)**
Unchanged from v2. Provenance tracking only.

**Step 2: parse_narrative() (Python)**
Unchanged. Sections, suspicious_total, review_period.

**Step 3: srl_extract() (Python)**
Unchanged. Lightweight supplement from Introduction + Conclusion.

**Step 4: extract_transactions() — LLM Call 1 (NEW)**

Scope: Details of Investigation section only
Prompt: Short, focused — read the transaction table and surrounding prose, return structured records

Output:
```json
{
  "transactions": [
    {
      "date": "10/14/2025",
      "direction": "inflow",
      "channel": "P2P",
      "amount": 250.00,
      "counterparty_type": "unknown"
    }
  ],
  "aggregate_pattern": {
    "dominant_inflow_channel": "P2P",
    "dominant_outflow_channel": "Cash",
    "min_amount": 10.00,
    "max_amount": 1200.00,
    "total_suspicious_amount": 25340.40,
    "transaction_count": 12,
    "review_period_days": 182,
    "is_self_to_self": true,
    "involves_hrc_jurisdiction": false,
    "velocity_flag": true
  }
}
```

Why LLM: Transaction table format inconsistent across narratives — C/D column present/absent, direction implicit/explicit in description, channel embedded in free text. LLM handles format variation naturally. Deterministic parser would be brittle.

Why aggregate_pattern: Individual transactions too granular for Rules Hypothesis. Aggregate captures the behavioral signature Agent 2 evaluates. Velocity flag captures temporal signals ("rapidly," "immediately") that amounts alone do not capture.

Why counterparty_type: HRC vs unknown vs self vs business is a critical AML typology dimension. P2P inflow from HRC counterparty is fundamentally different from self-to-self P2P.

Critical considerations for transaction extraction:
- Direction inference: sometimes explicit (C/D column), sometimes implicit from description keywords (WITHDRAWAL, Zelle Dep, Outgoing check)
- Channel inference from description: "International wire from X" → Wire Transfer, "Zelle Self-to-Self" → P2P, "WITHDRAWAL BRANCH" → Cash/ATM
- Counterparty type classification: HRC vs unknown vs known vs self vs business
- Exculpatory language: narrative describes both suspicious and normal activity — extract only suspicious transactions
- Aggregate summaries: "32 P2P Zelle deposits ranging from $10 to $1,000 totaling $27,000" more common than listing all rows
- Multiple accounts: narrative may cover primary account and related accounts — track which account each transaction belongs to
- Temporal velocity: "rapidly utilized," "immediately," "within days" — behavioral signals not captured by amounts alone

**Step 5: extract_behavioral_signals() — LLM Call 2 (NEW, replaces match_indicators)**

Scope: Introduction + Conclusion (primary), Details of Investigation prose excluding transaction table (secondary)

Prompt: "Describe the suspicious behavioral patterns observed in this narrative as a list of concise, generalizable signal statements. No proper nouns, no case-specific details, no dollar amounts, no country names."

Output:
```json
[
  "excessive self-to-self P2P transfers with no known source of funds",
  "funds rapidly moved through account with no legitimate purpose",
  "customer employment does not support level of financial activity",
  "counterparty relationships could not be established"
]
```

Why free-form instead of vocabulary matching: LLM is much better at reading a narrative and describing what it observes than at selecting from 150+ items. Free-form description captures the actual pattern. Over-matching eliminated because LLM is no longer choosing from a list.

Why exclude transaction table: Handled by Step 4. Step 5 focuses on investigator prose interpretation — contextual signals not in transaction records: employment mismatch, prior investigations, relationship history, derogatory information, business purpose analysis.

**Step 6: match_vocabulary() — Python Cosine Similarity (NEW, replaces LLM matching)**

Input: Behavioral signals from Step 5, full indicator vocabulary
Process: Compute cosine similarity between each behavioral signal and each vocabulary indicator using sentence embeddings. Return matches above similarity threshold, ranked by score.

Output:
```json
[
  {
    "behavioral_signal": "excessive self-to-self P2P transfers with no known source of funds",
    "matched_indicator_id": "FFI_F_FT_006",
    "similarity_score": 0.87
  },
  {
    "behavioral_signal": "customer employment does not support level of financial activity",
    "matched_indicator_id": "FFI_F_TFIN_002",
    "similarity_score": 0.91
  }
]
```

Why Python: Deterministic, reproducible, auditable. Threshold tunable. Can inspect exactly why a signal matched. Behavioral signals are already in clean generalizable language — semantically close to vocabulary text by design.

Why this eliminates over-matching: Cosine similarity is precise — "excessive P2P transfers with unknown source" scores high against FT/P2P indicators and low against structuring or round-tripping indicators. LLM thematic association eliminated.

**Step 7: Append to raw_indicator_matches.json (Python, updated schema)**
```json
{
  "case_id": "...",
  "narrative_date": "...",
  "active_fincen_tags": [...],
  "suspicious_total": 25340.40,
  "review_period": {"start": "...", "end": "..."},
  "spacy_frames": {...},
  "transaction_extraction": {
    "transactions": [...],
    "aggregate_pattern": {...}
  },
  "behavioral_signals": [...],
  "matched_indicators": [
    {
      "behavioral_signal": "...",
      "indicator_id": "...",
      "similarity_score": 0.87
    }
  ]
}
```

### consolidate_hypothesis() — Phase 2 (Python only, fully specified for v3)

Input: Complete raw_indicator_matches.json

Process:
1. Group matched_indicators by indicator_id across all records
2. Count frequency — frequency_pct = count / narratives_processed
3. Derive dominant channel from aggregate_pattern.dominant_inflow_channel across supporting cases
4. Derive direction from aggregate_pattern
5. Compute threshold as min(suspicious_total) across supporting cases — empirically grounded, not expert-set
6. Compute temporal_constraint as median(review_period_days) across supporting cases
7. Generate rule_ids in Python — guaranteed unique
8. Aggregate quantitative_pattern from transaction_extraction across supporting cases
9. Build Rules Hypothesis JSON

Output per rule:
```json
{
  "rule_id": "ML_821_001",
  "condition": "transaction.channel == 'P2P' AND transaction.direction == 'inflow'",
  "threshold": "$250.00",
  "temporal_constraint": "182 days",
  "indicators": [
    {
      "indicator_id": "FFI_F_FT_006",
      "indicator_text": "...",
      "source": "FFIEC_APPENDIX_F",
      "frequency": 7,
      "frequency_pct": 0.70,
      "avg_similarity_score": 0.89
    }
  ],
  "supporting_case_count": 10,
  "supporting_totals": [25340.40, 4573.31],
  "quantitative_pattern": {
    "dominant_inflow_channel": "P2P",
    "dominant_outflow_channel": "Cash",
    "typical_transaction_count": 12,
    "amount_range": {"min": 10.00, "max": 1200.00},
    "velocity_flag_pct": 0.70,
    "self_to_self_pct": 0.60,
    "hrc_involvement_pct": 0.30
  }
}
```

### Key Design Decisions Specific to v3

**Thresholds are empirically derived, not expert-set:**
min(suspicious_total) across supporting cases gives a data-grounded floor. SME thresholds excluded — they are arbitrary, population-non-specific, and import expert bias into a data-driven discovery system.

**SME rules excluded from input, reserved for post-hoc validation:**
SME rules tell you what investigators looked for. This system tells you what the data shows about filed cases. Differences between the two are the novel findings. SME rules used only as benchmark after consolidation, not as input.

**Severity estimation deferred:**
Severity (why some cases trigger 5 FinCEN tags vs 1) requires comparison of filed vs not-filed cases — a separate future agent. Out of scope for TCA v3.

**Channel granularity deferred:**
Current placeholder: BOT canonical channel values from spaCy and transaction extraction. Granularity review against AML vendor methodologies pending.

---

## Summary Comparison Table

| Dimension | v1 | v2 | v3 |
|---|---|---|---|
| LLM calls per narrative | 2 per tag | 1 | 2 (transaction + behavioral) |
| FinCEN tag role | Input filter | Provenance only | Provenance only |
| Rule assembly | LLM | Python | Python |
| Vocabulary | SAR procedures only | FFIEC F+G + SAR procedures | FFIEC F+G + SAR procedures |
| Indicator matching | LLM free synthesis | LLM picks from list | Python cosine similarity |
| Transaction extraction | None | None | LLM Call 1 (Details of Investigation) |
| Behavioral extraction | LLM structured extraction | LLM vocabulary matching | LLM free-form description |
| Threshold source | LLM (arbitrary) | Python min() — mostly null | Python min(suspicious_total) |
| Rules Hypothesis build | Incremental per narrative | Post-corpus Phase 2 | Post-corpus Phase 2 |
| Quantitative BOT conditions | No | No | Yes — from aggregate_pattern |
| Over-matching | Severe | Moderate | Eliminated by design |
| Hallucinated indicator IDs | Yes | Occasionally | Eliminated — Python matching |
| Agent 2 readiness | No | No | Yes |

---

## Files Changed Across Versions

| File | v1 | v2 | v3 |
|---|---|---|---|
| agent.py | induct() 2 LLM calls per tag | induct() Phase 1 + Phase 2 stub | induct() 4-step + consolidate_hypothesis() |
| llm_helpers.py | extract_narrative_structure() + synthesize_rules() | match_indicators() | extract_transactions() + extract_behavioral_signals() |
| sar_procedures.py | get_procedures_for_tags() | + HIGH_RISK_COUNTRIES + get_indicator_vocabulary() | + match_vocabulary() |
| parse_narrative.py | Unchanged | Unchanged | Unchanged |
| srl_extract.py | Unchanged | Unchanged | Unchanged |
| ffiec_appendix_f.json | Does not exist | Created verbatim | Unchanged |
| ffiec_appendix_g.json | Does not exist | Created | Unchanged |
| raw_indicator_matches.json | Does not exist | Phase 1 output | Extended schema |
| rules_hypothesis.json | Phase 1 incremental output | Phase 2 output (not finalized) | Phase 2 output (fully specified) |
| induct_tester.py | limit=3 notebook | limit=10 script | Updated for v3 pipeline |
| narrative_inspector.py | Does not exist | Created for v2 debugging | Unchanged |
