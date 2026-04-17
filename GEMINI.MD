# Typology Classification Agent (TCA) — Project Context

## Project Overview

The Typology Classification Agent (TCA) is a GCP/Vertex AI-based AI agent
being developed at KeyBank within the Financial Crimes Risk Management team.
It is Phase 1 of a two-phase AML multi-agent SAR automation system. The
agent's purpose is to learn to classify AML typologies from historical SAR
narratives with ground truth FinCEN tags, and then apply that learned Rules
Hypothesis to classify unseen SAR narratives.

Phase 2 is the BOT Package Agent, which evaluates raw transaction data from
Hogan Transaction BOT packages against the inducted Rules Hypothesis to
produce a binary escalation decision.

---

## Business Context

- **Team**: Financial Crimes Risk Management, KeyBank
- **Developer**: Emma Chen (edward_chen@keybank.com)
- **Environment**: Vertex AI Workbench (GCP), n2-standard-64, Gemini CLI
  (local Windows), VS Code
- **Primary language**: Python
- **LLM**: Gemini 2.5 Flash via Google ADK (google-adk)
- **Orchestration**: Google ADK (Agent Development Kit)
- **Data warehouse**: BigQuery (project: esa-src-prod-5736)
- **Observability**: Braintrust (not yet active — stubbed with DummySpan)
- **Proxy**: vzproxy.keybank.com (required for all pip installs)
- **GCP Project for Vertex AI**: taw-rmafincrime-prod-1398
- **GCP Location**: us-central1

---

## Critical Environment Details

### ADK Authentication
- Uses `.env` at `/home/jupyter/custom_agent/typology_agent/.env`
- Contents:
  GOOGLE_GENAI_USE_VERTEXAI=1
  GOOGLE_CLOUD_PROJECT=taw-rmafincrime-prod-1398
  GOOGLE_CLOUD_LOCATION=us-central1
- Model string: `gemini-2.5-flash`

### Installed NLP Packages (as of 2026-04-17)
- spaCy 3.8.14 — installed via proxy ✅
- en_core_web_sm — English model installed ✅
- torch — NOT installed (too large for proxy)
- transformers — NOT installed
- allennlp — NOT installed
- Decision: Use spaCy NER + noun chunk extraction instead of
  full SRL model. Gemini handles semantic enrichment in LLM calls.

---

## Architecture Overview

### induct() — Full Architecture

**Purpose:** Learn behavioral rules from labeled narratives (known FinCEN tags)

**Pipeline per narrative:**

Step 1: Pre-parse (Python, no LLM)
  parse_narrative(narrative)
  → suspicious_total: float  [regex on Introduction + Conclusion]
  → review_period: {start, end}  [regex on Introduction]

Step 2: spaCy Extraction (no LLM)
  srl_extract(narrative)
  → amounts: list[str]         [MONEY NER entities, deduplicated]
  → dates: list[str]           [DATE NER entities, filtered]
  → persons: list[str]         [PERSON NER entities, blocklisted]
  → channels: list[str]        [keyword match, longest-first]
  → transaction_types: list[str] [noun chunks, root-deduplicated]
  → red_flags: list[str]       [phrase match, canonical]

Step 3: SAR Procedures Lookup (Python, no LLM)
  get_procedures_for_tags(active_fincen_tags)
  → relevant_red_flags: {typology: [red flag strings]}
  [Selective — only sections for active tags, ~300-500 tokens per tag]

Step 4: Call 1 — Structured Extraction (LLM)
  System: EXTRACT_NARRATIVE_PROMPT + relevant_red_flags
  User: spaCy frames + raw narrative
  Role: Validation and enrichment of spaCy output, not open-ended
        extraction. Fills Details of Investigation and Suspicious
        Activity List sections.

Step 5: Call 2 — Rule Synthesis (LLM)
  System: SYNTHESIZE_RULES_PROMPT + relevant_red_flags
  User: spaCy frames + pre_parse (suspicious_total, review_period)
        + Call 1 output + existing rules hypothesis
  Role: Synthesize rules grounded by:
        - spaCy frames → condition field (observable events)
        - pre_parse → threshold + temporal_constraint (quantitative)
        - SAR procedures → indicators list (KeyBank red flag language)
        - existing_rules → merging and consolidation

Step 6: Post-processing (Python, no LLM)
  validate_rules_hypothesis(raw_output, active_fincen_tags,
                            narratives_processed)
  → tag completeness check → targeted retry if missing
  → empty rules array check → flag or retry
  → confidence computed mathematically:
     confidence = supporting_case_count / narratives_processed
  → validated_rules_hypothesis

### classify() — Full Architecture

**Purpose:** Classify unseen narrative. No ground truth tags available.

Step 1: Pre-parse (Python, no LLM) — same as induct()

Step 2: spaCy Extraction (no LLM) — same as induct()

Step 3: Rule Matching — Primary Signal (Python, no LLM)
  For each rule in Rules Hypothesis, evaluate whether spaCy frames
  satisfy rule condition. Returns ranked rule matches with scores.
  Fully deterministic — MRM can inspect exactly which frame
  triggered which rule.

Step 4: Embedding Similarity — Backup Signal (no LLM)
  Embed narrative text + rule indicators lists.
  Cosine similarity per rule. Catches cases where spaCy missed
  behavioral pattern due to unusual phrasing.
  Weight: 0.3 (vs 0.7 for rule matching)

Step 5: Candidate Tag Selection (Python, no LLM)
  Combine rule matching + embedding scores.
  Threshold at 0.3 to produce candidate typology list.

Step 6: SAR Procedures Lookup (Python, no LLM)
  get_procedures_for_tags(candidate_tags)
  [Selective injection based on candidate tags, not ground truth]

Step 7: Call 1 — Classification (LLM)
  System: CLASSIFY_NARRATIVE_PROMPT + relevant_red_flags
  User: candidate_tags_with_scores + matched_rules + spaCy frames
        + pre_parse + full narrative
  Role: Confirm or reject programmatically-identified candidates.
        NOT open-ended typology detection.

Step 8: Call 2 — Escalation Decision (LLM)
  System: ESCALATION_PROMPT
  User: confirmed_tags + scores + suspicious_total + matched_rules
        + narrative summary from Call 1
  Output: binary escalation flag + typology candidates + justification

Step 9: Post-processing (Python, no LLM)
  → escalation_flag: bool
  → typology_candidates: [{fincen_tag, confidence, supporting_rules}]
  → audit_trail: {srl_frames, rules_matched, embedding_scores,
                  llm_confirmation}

---

## Rules Hypothesis Schema (current)

```json
{
  "version": "v1",
  "typology_rules": [
    {
      "typology": "MONEY_LAUNDERING",
      "fincen_tag": "821",
      "confidence": 0.0,
      "rules": [
        {
          "rule_id": "ML_821_001",
          "condition": "transaction.channel == 'Cash' AND
                        transaction.direction == 'credit'",
          "threshold": "$X over Y days",
          "temporal_constraint": "N days/months",
          "indicators": [
            "unknown source of funds",
            "cash deposits inconsistent with income"
          ],
          "supporting_case_count": 0
        }
      ]
    }
  ],
  "provenance": {
    "narratives_processed": 0,
    "reading_plan_version": "v1"
  }
}
```

**Key schema design decisions:**
- `condition`: ONLY observable transaction fields. No narrative text.
  Machine-parseable for Agent 2 (BOT package) rule evaluation.
- `threshold`: minimum suspicious amount floor, not sum/average.
- `indicators`: narrative red flag language, separate from condition.
  Grows by adding genuinely new indicators only.
- `confidence`: computed mathematically post-induction.
  Formula: supporting_case_count / narratives_processed
- `supporting_case_count`: increments on merge, never resets.

**Rule ID format:** PREFIX_TAG_sequence
  ML = MONEY_LAUNDERING
  OSA = OTHER_SUSPICIOUS_ACTIVITY
  STR = STRUCTURING
  FRD = FRAUD
  TF = TERROR_FINANCING

---

## File Structure
/home/jupyter/custom_agent/typology_agent/
├── main.py
├── config.yaml
├── requirements.txt
├── CLAUDE.md
├── GEMINI.md
├── .env
├── tca/
│   ├── agent.py               # TypologyClassificationAgent
│   │                          # induct(), classify()
│   ├── app_config.py          # AppConfig frozen dataclass
│   │                          # includes target_fincen_tag field
│   ├── gr_config.py           # GoldenRecord (needs TCA update)
│   ├── llm_helpers.py         # All Gemini calls:
│   │                          #   generate_reading_plan()
│   │                          #   extract_narrative_structure()
│   │                          #   synthesize_rules()
│   │                          #   classify_narrative() [not yet built]
│   ├── parse_narrative.py     # Pre-LLM regex extraction:
│   │                          #   suspicious_total, review_period,
│   │                          #   sar_factors, keyword_matches
│   ├── srl_extract.py         # spaCy NER + noun chunk extraction
│   │                          # STATUS: rewrite in progress (see below)
│   ├── typology_keywords.py   # Domain dictionaries (SME-maintained):
│   │                          #   TYPOLOGY_KEYWORDS
│   │                          #   SAR_FACTORS_PATTERNS
│   │                          #   SUSPICIOUS_TOTAL_PATTERNS
│   │                          #   REVIEW_PERIOD_PATTERNS
│   │
│   └── tools/
│       ├── sar_narrative_tools.py  # get_sar_narratives()
│       │                           # Filters: SAR Monitoring Cases
│       │                           # excluded (B.SARMON = 0)
│       └── fincen_tag_tools.py     # get_fincen_tags()
│                                   # Filters: target_fincen_tag
│                                   # post-filter on all tag columns
└── evaluation/
├── row_based_eval_pipeline.py
├── configs/
│   ├── main_config.py
│   ├── eval_config.py      # NOT YET UPDATED for TCA
│   └── model_config.py     # NOT YET UPDATED for TCA
└── helpers/
├── braintrust_helpers.py
├── eval_helpers.py
├── querying_helpers.py
└── metric_support/

---

## Key Data Sources

### SAR Narratives (sar_narrative_tools.py)
- Function: get_sar_narratives(start_date, end_date, case_id=None)
- BQ Project: esa-src-prod-5736
- SAR Monitoring Cases excluded via: AND B.SARMON = 0 in WHERE clause
- Status: TESTED WORKING

### FinCEN Tags (fincen_tag_tools.py)
- Function: get_fincen_tags(start_date, end_date, case_id=None,
                            target_fincen_tag=None)
- target_fincen_tag post-filter: checks all tag columns
  (MONEY_LAUNDERING, OTH_SUS_ACTIVITY, STRCTNG, etc.) for presence
  of target sub-code string
- Current target: "821" (configurable via config.yaml)
- Status: TESTED WORKING

### Config (config.yaml)
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
limits:
  max_doc_chars_for_llm: 90000
  max_rows_per_query: 2000
  max_rows_in_response: 200
```

---

## srl_extract.py — Current Status and Stopping Point

**Status: Partially working. Rewrite in progress.**

### What works:
- spaCy en_core_web_sm installed and loading correctly
- MONEY entity extraction — correctly captures dollar amounts
- Channel keyword matching — correctly identifies Zelle, Cash, Check etc.
- Red flag phrase matching — correctly identifies AML indicators
- Section splitting (Introduction + Conclusion) working

### Known issues being fixed in rewrite:
1. Account numbers captured as DATE entities — fix: filter purely
   numeric strings from DATE entities
2. Amounts and transaction types duplicated — fix: deduplication
   by root word, keep longest chunk per root
3. Person entities inconsistent — "Zelle" captured as PERSON — fix:
   PERSON_BLOCKLIST of known non-person terms
4. Transaction types too generic — fix: require informative keyword,
   filter GENERIC_ACTIVITY_PHRASES set

### Rewrite approach:
Switched from verb-argument extraction to NER + noun chunk analysis.
SAR Introduction sentences use nominalized transactional language
("significant cash deposits") not active voice ("deposited cash"),
so dependency parsing on main verbs misses key information.
NER correctly captures MONEY and DATE. Noun chunks correctly capture
transaction type phrases.

### Next step when resuming:
Replace srl_extract.py with the complete rewritten version
(provided at end of previous session). Run tester on 3 cases:
- AML_INV_00000051420 (Joan Gaylord — cash deposit, tag 821)
- AML_INV_00000055001 (Brent Hemphill — OBC funnel, tags 821/824)
- AML_INV_00000049396 (Ylionet Doreus — Zelle P2P, tags 805/821)

Evaluate output quality before wiring srl_extract into agent.py.

---

## Prompts — Current State

### SYNTHESIZE_RULES_PROMPT (llm_helpers.py lines 273-325)
Current version includes:
- One typology_rules entry per fincen_tag requirement
- Merging rules (same fincen_tag + same primary channel = merge)
- Confidence scoring (1.0/0.7/0.5) — TO BE REPLACED with
  mathematical computation post-induction
- Rule ID format: PREFIX_TAG_sequence
- condition field: observable transaction fields only, no narrative
- indicators field: separate list of red flag phrases
- threshold: minimum suspicious floor, never sum/average
- Constraint: no OR in condition field

### EXTRACT_NARRATIVE_PROMPT (llm_helpers.py lines 158-243)
Current schema extracts:
- introduction: {alert_trigger, review_period_start/end,
                 accounts_involved, total_suspicious_amount, summary}
- details_of_investigation: {customer_profile, account_summaries[]}
- suspicious_activity_list: [{date, amount, transaction_type,
                               channel, counterparty, direction}]
- conclusion: {summary, total_suspicious_amount, key_findings[]}
- behavioral_components: {transaction_patterns[], dollar_thresholds[],
                          time_window_days, account_types[],
                          channels[], geographic_risk_flags[],
                          counterparty_relationship,
                          velocity_indicators[], layering_indicators[]}

**Planned update:** Add spaCy pre-extracted signals to user prompt
as grounding context before narrative text.

---

## What's Working vs Pending

### WORKING ✅
- get_sar_narratives() — SAR Monitoring Cases excluded
- get_fincen_tags() — target_fincen_tag filter working
- app_config.py + config.yaml — target_fincen_tag field added
- agent.py — FINCEN_FLAG_KEYS + TYPOLOGY_LABEL_MAP expansion
- active_fincen_tags filtering — 999/9999 codes excluded
- parse_narrative.py — suspicious_total + review_period regex
- srl_extract.py — partial (see above)
- typology_keywords.py — TYPOLOGY_KEYWORDS, regex patterns
- induct() pipeline — end-to-end proven, 3-10 narratives
- Rules Hypothesis — condition/indicators/threshold schema working
- SAR Monitoring Case filter in sar_narrative_tools.py

### PENDING ⏳
- srl_extract.py rewrite — complete file written, not yet tested
- Wire srl_extract into agent.py induct() pipeline
- sar_procedures.py — SAR procedures manual chunked by typology
  (manual available, pages 10-28 not yet photographed/transcribed)
- get_procedures_for_tags() — selective injection function
- Update EXTRACT_NARRATIVE_PROMPT to use spaCy grounding
- Update SYNTHESIZE_RULES_PROMPT to use spaCy + pre_parse grounding
- Remove confidence from LLM output, compute mathematically in
  validate_rules_hypothesis()
- validate_rules_hypothesis() — post-processing validator
- classify() — not yet built
- gr_config.py — still CRA version
- evaluation/configs/ — still CRA version
- Braintrust — access not provisioned
- GitLab — access not sorted

---

## Immediate Next Steps (priority order)

1. Replace srl_extract.py with complete rewrite
2. Run tester on 3 cases, evaluate output quality
3. Wire srl_extract into agent.py — call before extract_narrative_structure()
4. Update EXTRACT_NARRATIVE_PROMPT to inject spaCy frames
5. Update SYNTHESIZE_RULES_PROMPT to inject spaCy frames + pre_parse
6. Remove confidence from LLM, compute in post-processing
7. Build validate_rules_hypothesis()
8. Build sar_procedures.py from SAR procedures manual
9. Run induct() end-to-end with full new architecture at limit=10
10. Build classify()

---

## Agent 2 — BOT Package Agent (Future)

**Status: Deferred. Design only.**

Purpose: Evaluate raw transaction data from Hogan Transaction BOT
packages against inducted Rules Hypothesis. Produce binary escalation
decision without SAR narrative input.

Key differences from Agent 1:
- No narrative text — no spaCy, no NER, no prose parsing needed
- No summary statistics — all features computed from raw transaction rows
- Rule matching is deterministic numerical comparison, not semantic
- condition field in Rules Hypothesis must be machine-parseable SQL-like
  syntax (this is why condition design matters in Agent 1)
- Vendor transaction rules available as grounding (deferred)
- One LLM call (not two) — receives rule evaluation results + feature
  summary, produces escalation decision with justification

Feature engineering layer will be substantial:
- Transaction velocity by channel over configurable time windows
- Credit/debit ratios
- Counterparty diversity metrics
- Threshold comparison against Rules Hypothesis per rule

---

## Key Design Decisions and Principles

### Pipeline vs Agent
- induct() is a pipeline — sequential, no tool use, deterministic steps
- classify() will have more agentic characteristics — candidate selection,
  conditional retry, multi-signal fusion

### Confidence is Mathematical
- LLM does NOT set confidence
- confidence = supporting_case_count / narratives_processed
- Laplace smoothing at scale: (count + 1) / (narratives_processed + 2)

### Condition Field is Machine-Parseable
- Must evaluate programmatically against BOT package transaction data
- Format: transaction.field == 'value' AND transaction.field >= X
- No narrative prose in condition field ever
- OR not permitted — use separate rules for distinct channels

### Grounding Architecture (three independent sources)
1. spaCy NER + noun chunks — deterministic behavioral signals
2. parse_narrative regex — quantitative anchors (amount, dates)
3. SAR procedures manual — KeyBank domain knowledge (selective)

### SAR Procedures Grounding (selective injection)
- NOT full document in system prompt (10 pages = too large)
- Pre-chunked by typology in sar_procedures.py
- At call time: inject only sections for active FinCEN tags
- ~300-500 tokens per tag, 2-3 tags typical = 600-1500 tokens
- Goes in system prompt of both Call 1 and Call 2

### target_fincen_tag
- Currently: "821" for diagnostic single-typology induction
- Configured in config.yaml defaults.target_fincen_tag
- Set to null/empty for multi-typology induction
- Filter applied in get_fincen_tags() post-query
- Includes narratives where tag is present alongside others
  (e.g. 821+805 narrative is included when target=821)

### SAR Monitoring Cases Excluded
- Filtered in sar_narrative_tools.py WHERE clause
- Condition: AND B.SARMON = 0
- Rationale: monitoring case narratives have different structure
  and writing style — not suitable for induction

---

## Tools and Infrastructure

- LLM: Google Gemini 2.5 Flash
- Infrastructure: GCP, Vertex AI, BigQuery, GCS
- Observability/Eval: Braintrust (stubbed)
- NLP: spaCy 3.8.14 + en_core_web_sm
- Libraries: google-adk, google-genai, spaCy, pandas, rdflib
- Data: BigQuery (SAR narratives, FinCEN tags, MDM)
