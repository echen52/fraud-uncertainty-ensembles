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

Step 1: Build active_fincen_tags (Python, no LLM)
  Constructed from BigQuery row data at top of loop.
  Filters out 999/9999 catch-all codes.
  Must run BEFORE all other steps so procedures can be injected.

Step 2: Pre-parse (Python, no LLM)
  parse_narrative(narrative)
  → sections: {introduction, details_of_investigation,
               conclusion, contents_of_file}
  → suspicious_total: float  [regex on Conclusion + Introduction]
  → review_period: {start, end}  [regex on Introduction]

Step 3: spaCy Extraction (no LLM)
  srl_extract(narrative)
  → amounts: list[str]           [MONEY NER, deduplicated, account
                                  numbers filtered >12 digits]
  → channels: list[str]          [keyword match, longest-first]
  → transaction_types: list[str] [noun chunks, root-deduplicated]
  → red_flags: list[str]         [phrase match, canonical]

Step 4: SAR Procedures Lookup (Python, no LLM)
  get_procedures_for_tags(active_fincen_tags)
  → relevant_red_flags: formatted string of KeyBank indicator text
  [Selective — code-level refinement takes priority over group-level.
   ~300-500 tokens per tag, 2-3 tags typical = 600-1500 tokens]

Step 5: Call 1 — Structured Extraction (LLM)
  System: EXTRACT_NARRATIVE_PROMPT
    — describes five-section SAR structure
    — treats spaCy signals as ground truth anchors
    — instructs on suspicious activity list ambiguity
  User: spaCy frames + suspicious_total + review_period
        + SAR procedures + pre-split narrative sections
  Role: Validate and enrich spaCy signals. Extract structured
        object from pre-split sections. Fill details of
        investigation and suspicious activity list.

Step 6: Call 2 — Rule Synthesis (LLM)
  System: SYNTHESIZE_RULES_PROMPT
    — merging rules (same fincen_tag + same primary channel)
    — grounding rules (condition/threshold/indicators grounded
      by spaCy frames + pre_parse)
    — output rules (permitted condition fields + BOT package
      canonical channel values)
    — confidence always 0.0 (computed mathematically post-induction)
  User: spaCy frames + pre_parse + SAR procedures
        + extracted structure + active_fincen_tags
        + existing rules hypothesis
  Role: Synthesize rules grounded by all three sources.
        Merge with existing rules where same tag + channel.

Step 7: Post-processing (Python, no LLM) — PENDING
  validate_rules_hypothesis() — not yet built

### classify() — NOT YET BUILT

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
                        transaction.direction == 'inflow'",
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
- `condition`: ONLY three permitted fields:
  - `transaction.channel` — BOT package canonical values only:
    'P2P', 'Cash', 'Wire Transfer', 'ACH', 'Check', 'ATM'
    Platform-specific names (Zelle, Cash App, NFCU P2P) map to 'P2P'
    and belong in indicators only, never in condition.
  - `transaction.direction` — inflow/outflow/credit/debit
  - `transaction.amount` — dollar amount comparison
  - OR permitted only for functionally equivalent channels.
    P2P covers all peer-to-peer platforms — OR never needed for P2P.
- `threshold`: minimum suspicious amount floor, not sum/average.
- `indicators`: narrative red flag language, generalizable across cases.
  No case-specific details (country names, account numbers, prior
  investigation history). Grows by adding genuinely new indicators only.
- `confidence`: always 0.0 from LLM. Computed mathematically post-induction.
  Formula: supporting_case_count / narratives_processed
- `supporting_case_count`: increments on merge, never resets.

**Rule ID format:** PREFIX_TAG_sequence
  ML = MONEY_LAUNDERING
  OSA = OTHER_SUSPICIOUS_ACTIVITY
  STR = STRUCTURING
  FRD = FRAUD
  TF = TERROR_FINANCING

---

## Three Grounding Sources

All three run deterministically before any LLM call:

1. **parse_narrative.py** — section splitter + quantitative anchors
   - Splits narrative into 4 reliable sections via regex on ALL CAPS headers
   - INTRODUCTION, DETAILS OF INVESTIGATION, CONCLUSION, CONTENTS OF FILE
   - Suspicious Activity List has no reliable header — embedded in Details
     of Investigation as a formatted list. Left for LLM to identify.
   - Extracts suspicious_total (regex, Conclusion priority)
   - Extracts review_period start/end (regex, Introduction priority)

2. **srl_extract.py** — spaCy NER + noun chunk extraction
   - Runs on Introduction + Conclusion only (not full narrative)
   - Returns: amounts, channels, transaction_types, red_flags
   - dates and persons removed (dates handled by parse_narrative,
     persons not useful as behavioral signal)
   - Account number filter: MONEY entities with >12 raw digits rejected
   - format_frames_for_prompt() formats output for LLM injection

3. **sar_procedures.py** — KeyBank SAR procedure grounding
   - Loads tca/data/sar_procedure.json at module level
   - FINCEN_CODE_MAP: numeric code → typology group + description
   - SAR_PROCEDURES_INDEX: typology group → relevant section numbers
   - FINCEN_CODE_SECTION_REFINEMENT: code-level section override
     (e.g. 821 → sections 1,2,5,16 only, not all ML sections)
   - get_procedures_for_tags(active_fincen_tags) → formatted string
   - Deduplicates sections across multiple tags of same typology group

---

## SAR Narrative Structure

Five formal sections, always present, always ALL CAPS headers
(exception: Suspicious Activity List):

1. INTRODUCTION — alert trigger, review period, accounts involved,
   total suspicious amount summary
2. DETAILS OF INVESTIGATION — customer profile, account summaries,
   transaction detail. Suspicious Activity List may be embedded here
   as a formatted list rather than its own labeled section.
3. SUSPICIOUS ACTIVITY LIST — may appear as own section or embedded
   within Details of Investigation.
4. CONCLUSION — summary of findings, total suspicious amount confirmed
5. CONTENTS OF FILE — supporting documents list, not useful for extraction

SAR Monitoring Cases excluded (SARMON = 0 filter in sar_narrative_tools.py)
— monitoring case narratives have different structure and writing style.

---

## FinCEN Code Mapping

Numeric codes map to typology groups:
- 1xx = Structuring (STRCTNG)
- 3xx = Fraud (FRAUD)
- 4xx = Identification/Documentation (IDNTFCTN)
- 5xx = Insurance (INSURANCE)
- 6xx = Securities/Futures/Options (SFO)
- 7xx = Terrorist Financing (TERROR_FINANCING)
- 8xx = Money Laundering (MONEY_LAUNDERING)
- 9xx = Other Suspicious Activity (OTH_SUS_ACTIVITY)
- 10xx = Mortgage Fraud (MORTGAGE)
- 11xx = Cyber Event (CYBER)
- 12xx = Gaming Activities (GAMING)
- x999/x9999 = Other catch-alls (filtered out)

Key Money Laundering codes (current focus):
- 821 = Suspicious concerning the source of funds
- 824 = Funnel account
- 805 = Suspicious EFT/wire transfers
- 812 = Transaction out of pattern for customer(s)
- 807 = Suspicious use of multiple accounts

Key Other Suspicious Activity codes:
- 925 = Transaction with no apparent economic, business, or lawful purpose
- 911 = Two or more individuals working together

---

## BOT Package Transaction Data

BOT package `Trans_Type_Code_Desc` field uses these canonical values
(relevant to Rules Hypothesis condition field):
- `P2P FUNDS TRANSFER DEPOSIT` / `P2P FUNDS TRANSFER WITHDRAWAL`
  — all P2P regardless of platform (Zelle, Cash App, Venmo etc.)
- `CUSTOMER WITHDRAWAL`
- `CREDIT-ACCOUNT TO ACCOUNT TRANSFER`
- `DEBIT-ACCOUNT TO ACCOUNT TRANSFER`

Note: BOT package does not distinguish between P2P platforms.
All peer-to-peer maps to 'P2P' in condition field.
Additional BOT package channel values to be confirmed as Agent 2
architecture develops.

---

## File Structure
/home/jupyter/custom_agent/typology_agent/
├── agent_tester.ipynb          # Main pipeline tester
├── cases_by_CASE_ID.ipynb      # Test specific case IDs
├── check_FINCEN_tag_tool.ipynb # FinCEN tag tool tester
├── check_SAR_tool.ipynb        # SAR narrative tool tester
├── config.yaml                 # Configuration
├── GEMINI.md                   # This file
├── main.py
├── parse_narrative_tester.ipynb
├── requirements.txt
├── srl_extract_tester.ipynb
├── evaluation/                 # NOT YET UPDATED for TCA
│                               # Still CRA version
└── tca/
├── init.py
├── agent.py                # TypologyClassificationAgent
│                           # induct() pipeline
├── app_config.py           # AppConfig frozen dataclass
├── llm_helpers.py          # All Gemini calls:
│                           #   extract_narrative_structure()
│                           #   synthesize_rules()
│                           #   classify_narrative() [not built]
├── parse_narrative.py      # Section splitter + regex anchors:
│                           #   split_narrative_sections()
│                           #   extract_suspicious_total()
│                           #   extract_review_period()
│                           #   parse_narrative() → main entry
├── sar_procedures.py       # SAR procedure grounding:
│                           #   FINCEN_CODE_MAP
│                           #   SAR_PROCEDURES_INDEX
│                           #   FINCEN_CODE_SECTION_REFINEMENT
│                           #   get_procedures_for_tags()
│                           #   get_code_description()
│                           #   get_code_typology_group()
├── srl_extract.py          # spaCy NER + noun chunks:
│                           #   srl_extract() → main entry
│                           #   format_frames_for_prompt()
├── typology_keywords.py    # Domain patterns:
│                           #   SUSPICIOUS_TOTAL_PATTERNS ✅ used
│                           #   REVIEW_PERIOD_PATTERNS ✅ used
│                           #   TYPOLOGY_KEYWORDS ⚠️ unused
│                           #   SAR_FACTORS_PATTERNS ⚠️ unused
├── data/
│   └── sar_procedure.json  # KeyBank SAR procedure manual
│                           # Sections 9-25, loaded by
│                           # sar_procedures.py at module level
├── not used/               # Legacy files from commercial agent
│                           # Safe to ignore
└── tools/
├── init.py
├── bq_tools.py         # Legacy from commercial agent
├── fincen_tag_tools.py # get_fincen_tags() ✅ active
├── gcs_tools.py        # Legacy from commercial agent
└── sar_narrative_tools.py # get_sar_narratives() ✅ active

---

## Key Data Sources

### SAR Narratives (sar_narrative_tools.py)
- Function: get_sar_narratives(start_date, end_date, case_id=None)
- BQ Project: esa-src-prod-5736
- SAR Monitoring Cases excluded via: AND B.SARMON = 0 in WHERE clause
- Status: TESTED WORKING ✅

### FinCEN Tags (fincen_tag_tools.py)
- Function: get_fincen_tags(start_date, end_date, case_id=None,
                            target_fincen_tag=None)
- target_fincen_tag post-filter: checks all tag columns
  (MONEY_LAUNDERING, OTH_SUS_ACTIVITY, STRCTNG, etc.) for presence
  of target sub-code string
- Current target: "821" (configurable via config.yaml)
- Status: TESTED WORKING ✅

### Closed No-SAR Cases (future — severity calibration agent)
- Cases with status "CLOSED" and no SAR filed exist in BigQuery
- Filtered similarly to SAR Monitoring Cases
- Not in scope for TCA — reserved for future severity agent
- Document location when needed: same BQ project, status field TBD

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

## What's Working vs Pending

### WORKING ✅
- get_sar_narratives() — SAR Monitoring Cases excluded
- get_fincen_tags() — target_fincen_tag filter working
- app_config.py + config.yaml — target_fincen_tag field added
- agent.py — induct() pipeline end-to-end proven at limit=3
- active_fincen_tags — built at top of loop before LLM calls
- parse_narrative.py — section splitter + suspicious_total
  + review_period regex
- srl_extract.py — amounts, channels, transaction_types, red_flags
  dates and persons removed. Account number filter working.
- sar_procedures.py — selective injection by FinCEN code working
- tca/data/sar_procedure.json — loaded at module level
- FINCEN_CODE_MAP — all codes mapped ✅
- FINCEN_CODE_SECTION_REFINEMENT — code-level section override ✅
- llm_helpers.py — extract_narrative_structure() + synthesize_rules()
  both receiving spaCy frames + pre_parse + procedures grounding
- Rules Hypothesis — condition/indicators/threshold schema working
  confidence = 0.0 enforced, transaction.type field removed
  BOT package canonical channel values enforced
  P2P normalization enforced in prompt
- SAR Monitoring Case filter in sar_narrative_tools.py

### PENDING ⏳
- Run induct() at limit=10 — evaluate consolidation quality
- High risk country list — add to sar_procedures.py or separate file
  KeyBank official list to be located. Indicators should reference
  category not specific country name.
- validate_rules_hypothesis() — post-processing validator
  tag completeness check, empty rules check, confidence computation
  Formula: supporting_case_count / narratives_processed
- Wire srl_extract into classify() pipeline (classify not yet built)
- classify() — not yet built
- Update EXTRACT_NARRATIVE_PROMPT to inject spaCy grounding
  (partially done — sections injected, grounding language in system)
- sar_procedures.py — FINCEN_CODE_SECTION_REFINEMENT needs review
  after limit=10 run to verify section relevance
- gr_config.py — still CRA version, in not used/
- evaluation/configs/ — still CRA version
- Braintrust — access not provisioned
- GitLab — access not sorted

---

## Immediate Next Steps (priority order)

1. Run induct() at limit=10, evaluate Rules Hypothesis consolidation
2. Add high risk country list after director touchbase confirms source
3. Build validate_rules_hypothesis() post-processing
4. Build classify()
5. Move to Gemini CLI — commit clean checkpoint to GitLab first

---

## Key Design Decisions and Principles

### Reading Plan — Removed
- Previously: LLM-generated reading plan from narrative sample
- Removed: unstable, LLM-generated content stacked on LLM calls,
  large token cost, replaced by better deterministic alternatives
- Replaced by: deterministic section splitter in parse_narrative.py
  + hardcoded structural description in EXTRACT_NARRATIVE_PROMPT

### Pipeline vs Agent
- induct() is a pipeline — sequential, no tool use, deterministic steps
- classify() will have more agentic characteristics — candidate
  selection, conditional retry, multi-signal fusion

### Confidence is Mathematical
- LLM does NOT set confidence — always 0.0 in output
- confidence = supporting_case_count / narratives_processed
- Laplace smoothing at scale: (count + 1) / (narratives_processed + 2)

### Condition Field is Machine-Parseable
- Must evaluate programmatically against BOT package transaction data
- Only three fields permitted: transaction.channel, transaction.direction,
  transaction.amount
- BOT package canonical channel values only in condition
- Platform-specific names (Zelle, NFCU P2P) in indicators only

### Severity Gap (future work)
- TCA learns WHAT patterns are suspicious, not HOW SEVERE
- Severity threshold learned from closed no-SAR cases vs filed SARs
- Separate future agent — not TCA scope
- Director confirmed: primary use case is ASM 2.0 feature engineering
  (red flags as candidate features for risk rank ordering)

### Grounding Architecture (three independent sources)
1. parse_narrative — section split + quantitative anchors
2. srl_extract — deterministic behavioral signals from intro/conclusion
3. sar_procedures — KeyBank domain knowledge (selective by tag)

### SAR Procedures Grounding (selective injection)
- NOT full document in system prompt
- Pre-chunked by typology in sar_procedures.py
- Code-level refinement: specific codes get specific sections
- At call time: inject only sections for active FinCEN tags
- Goes in user prompt of both Call 1 and Call 2

### target_fincen_tag
- Currently: "821" for diagnostic single-typology induction
- Configured in config.yaml defaults.target_fincen_tag
- Set to null/empty for multi-typology induction
- Filter applied in get_fincen_tags() post-query
- Includes narratives where tag is present alongside others

### SAR Monitoring Cases Excluded
- Filtered in sar_narrative_tools.py WHERE clause
- Condition: AND B.SARMON = 0
- Rationale: monitoring case narratives have different structure
  and writing style — not suitable for induction

---

## Agent 2 — BOT Package Agent (Future)

**Status: Deferred. Design only.**

Purpose: Evaluate raw transaction data from Hogan Transaction BOT
packages against inducted Rules Hypothesis. Produce binary escalation
decision without SAR narrative input.

Key differences from Agent 1:
- No narrative text — no spaCy, no NER, no prose parsing needed
- Rule matching is deterministic numerical comparison, not semantic
- condition field in Rules Hypothesis must be machine-parseable
  (this is why condition design matters in Agent 1)
- One LLM call (not two)

BOT package Trans_Type_Code_Desc known values (partial):
- P2P FUNDS TRANSFER DEPOSIT / WITHDRAWAL
- CUSTOMER WITHDRAWAL
- CREDIT-ACCOUNT TO ACCOUNT TRANSFER
- DEBIT-ACCOUNT TO ACCOUNT TRANSFER
Additional values to be confirmed during Agent 2 development.

---

## Tools and Infrastructure

- LLM: Google Gemini 2.5 Flash
- Infrastructure: GCP, Vertex AI, BigQuery, GCS
- Observability/Eval: Braintrust (stubbed)
- NLP: spaCy 3.8.14 + en_core_web_sm
- Libraries: google-adk, google-genai, spaCy, pandas
- Data: BigQuery (SAR narratives, FinCEN tags, MDM)
