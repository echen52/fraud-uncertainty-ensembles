# Typology Classification Agent (TCA) — Project Context v2

## Version History

- **v1** — Initial design. induct() used two LLM calls per narrative per FinCEN tag:
  Call 1 (structured extraction) and Call 2 (rule synthesis). LLM was responsible for
  synthesizing rule objects, merging rules across narratives, and managing the Rules
  Hypothesis schema. This produced indicator bloat, cross-tag contamination, case-specific
  entity leakage, and inconsistent thresholds. Abandoned after limit=10 evaluation.
- **v2** — Redesigned. LLM scope radically reduced to a single matching call per narrative
  against a controlled indicator vocabulary. All rule assembly, merging, thresholds, and
  schema construction moved to deterministic Python. Rules Hypothesis built in a single
  post-corpus consolidation pass, not incrementally.

---

## Project Overview

The Typology Classification Agent (TCA) is a GCP/Vertex AI-based AI agent being developed
at KeyBank within the Financial Crimes Risk Management team. It is Phase 1 of a two-phase
AML multi-agent SAR automation system.

The agent's purpose is to build a structured typology knowledge base from historical SAR
narratives by matching narrative evidence against a controlled indicator vocabulary derived
from KeyBank SAR procedures and FFIEC regulatory guidance. This knowledge base serves as
a candidate feature source for ASM 2.0 risk rank ordering.

Phase 2 is the BOT Package Agent, which evaluates raw transaction data from Hogan
Transaction BOT packages against the inducted Rules Hypothesis to produce a binary
escalation decision.

### What This System Is and Is Not

**Is:** Guided extraction of known AML indicator patterns from a SAR corpus against a
controlled regulatory vocabulary. Output is auditable, defensible, and grounded in
established typology frameworks.

**Is not:** Unsupervised pattern discovery or severity boundary learning. Discovering
hidden indicators or severity separation (filed vs. not-filed cases) requires a separate
future agent with access to closed no-SAR cases and is out of scope for TCA.

---

## Business Context

- **Team**: Financial Crimes Risk Management, KeyBank
- **Developer**: Emma Chen (edward_chen@keybank.com)
- **Environment**: Vertex AI Workbench (GCP), n2-standard-64, Gemini CLI (local Windows),
  VS Code
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
- Decision: Use spaCy NER + noun chunk extraction instead of full SRL model.
  Gemini handles semantic matching in the single LLM call.

---

## Architecture Overview — v2

### Design Principle

induct() is a **data pipeline with a single narrow LLM matching step**, not an agentic
system. The LLM answers one question per narrative: which items from the controlled
indicator vocabulary are evidenced in this narrative? All structure, assembly, and
aggregation is deterministic Python.

This follows the Microsoft/AWS guidance that nondeterministic LLM components should be
used at the lowest complexity level that reliably meets requirements. Free-form indicator
synthesis (v1) was a nondeterministic pattern applied to an inherently deterministic
extraction problem. v2 corrects this.

### induct() — Full Architecture v2

**Purpose:** Build a typology knowledge base by matching SAR narrative evidence against
a controlled indicator vocabulary across a corpus of historical cases.

**Two-phase structure:**

#### Phase 1: Per-Narrative Processing (LLM involved)

For each narrative in the corpus:

**Step 1: Build active_fincen_tags (Python, no LLM)**
Constructed from BigQuery row data. Filters out 999/9999 catch-all codes.
Used for output validation and provenance tracking, NOT as an input filter
for LLM extraction. Tags are the investigator's conclusion — the narrative
contains mixed evidence across all behaviors regardless of tag.

**Step 2: Pre-parse (Python, no LLM)**
parse_narrative(narrative)
→ sections: {introduction, details_of_investigation, conclusion, contents_of_file}
→ suspicious_total: float [regex on Conclusion + Introduction]
→ review_period: {start, end} [regex on Introduction]

**Step 3: spaCy Extraction (Python, no LLM)**
srl_extract(narrative)
→ amounts: list[str]
→ channels: list[str] [BOT canonical values only]
→ transaction_types: list[str]
→ red_flags: list[str]
→ direction: inflow/outflow/both [derived from spaCy frames]

**Step 4: Load Controlled Indicator Vocabulary (Python, no LLM)**
get_indicator_vocabulary(active_fincen_tags)
→ Merges relevant sections from:
  - tca/data/sar_procedure.json (KeyBank SAR procedures)
  - tca/data/ffiec_appendix_f.json (FFIEC ML/TF red flags)
  - tca/data/ffiec_appendix_g.json (FFIEC Structuring definitions)
→ Each indicator in the vocabulary has: indicator_id, category, text, source
→ Selective injection by typology relevance (same logic as v1 sar_procedures.py)

**Step 5: Single LLM Call — Indicator Matching**
System: MATCH_INDICATORS_PROMPT
- Describes the controlled vocabulary structure
- Instructs model to match only — never invent indicators not in the vocabulary
- Instructs model to return indicator_ids with one-sentence evidence quote
- No proper nouns, no case-specific details in evidence quotes
- No rule structure, no schema, no channel assignment — matching only

User: Narrative sections (Introduction + Conclusion primary,
      Details of Investigation secondary) + formatted indicator vocabulary

Returns: flat list of {indicator_id, evidence_quote} objects. Nothing else.

**Step 6: Append to Raw Indicator Matches (Python, no LLM)**
Appends to raw_indicator_matches.json:
{
  "case_id": "...",
  "narrative_date": "...",
  "active_fincen_tags": [...],
  "suspicious_total": float,
  "review_period": {start, end},
  "spacy_frames": {channels, direction, amounts, transaction_types, red_flags},
  "matched_indicators": [
    {"indicator_id": "...", "evidence_quote": "..."},
    ...
  ]
}

No merging. No rule assembly. Just append and continue.

#### Phase 2: Post-Corpus Consolidation (Python only, no LLM)

After all narratives are processed, a single Python consolidation pass builds
the Rules Hypothesis from raw_indicator_matches.json.

**consolidate_hypothesis() — Python only:**
- Groups matches by indicator_id
- Counts frequency of each indicator across corpus
- Derives channel from spaCy frames (BOT canonical values)

  ⚠️ PENDING: Channel granularity review required before this step is finalized.
  Current placeholder: use BOT canonical channel values from spaCy frames.
  May require refinement based on AML vendor methodology review.

- Derives direction from spaCy frames
- Computes threshold as min(suspicious_total) across all cases where indicator matched
- Assigns rule_id in Python (format: PREFIX_TAG_sequence, guaranteed unique)
- Computes confidence: frequency / narratives_processed
- Builds final Rules Hypothesis JSON

### classify() — NOT YET BUILT

Will have more agentic characteristics than induct() — candidate selection,
conditional retry, multi-signal fusion against Rules Hypothesis.

---

## Controlled Indicator Vocabulary — Structure

Three source files, all in tca/data/:

### sar_procedure.json (existing)
KeyBank internal SAR procedure manual. Sections 9-25.
Already structured and loaded by sar_procedures.py.
indicator_id format: KBP_{section}_{sequence}

### ffiec_appendix_f.json (NEW — to be built)
FFIEC BSA/AML Manual Appendix F — Money Laundering and Terrorist Financing Red Flags.
Source: https://bsaaml.ffiec.gov/manual/Appendices/07
Organized by category:
- Insufficient/Suspicious Customer Information
- Efforts to Avoid Reporting Requirements
- Funds Transfers
- ACH Transactions
- Activity Inconsistent with Customer's Business
- Lending Activity
- Structuring/Currency patterns
- Trade Finance
- Shell Company Activity
- Other Unusual Activity
- Terrorist Financing indicators

indicator_id format: FFI_F_{category_code}_{sequence}
Each entry: {indicator_id, category, text, source: "FFIEC_APPENDIX_F"}

### ffiec_appendix_g.json (NEW — to be built)
FFIEC BSA/AML Manual Appendix G — Structuring.
Source: https://bsaaml.ffiec.gov/manual/Appendices/08
Narrower scope — structuring-specific definitions and patterns.
indicator_id format: FFI_G_{sequence}
Each entry: {indicator_id, category, text, source: "FFIEC_APPENDIX_G"}

---

## High-Risk Country List

Stored as HIGH_RISK_COUNTRIES constant in sar_procedures.py.
Source: KeyBank Financial Crimes Risk Management SharePoint (official list).

Countries:
Afghanistan, Belarus, Central African Republic,
Congo (Democratic Republic of), Cuba, Ethiopia, Iran, Iraq, Libya, Mali,
Myanmar (Burma), Nicaragua, North Korea (Democratic People's Republic of Korea),
Russia (Russian Federation), Somalia, Sudan, South Sudan,
Syria (Syrian Arab Republic),
Ukraine (Crimea Region, Donetsk People's Republic, Luhansk People's Republic),
Venezuela, Yemen

Note: The Balkans excluded per KeyBank policy review.
Note: China and Mexico are NOT on this list and must not be treated as HRC indicators.
Any country NOT on this list appearing in a narrative = foreign counterparty only,
not an HRC indicator.

In evidence quotes: any HRC country name → "high-risk foreign jurisdiction".
No country name of any kind should appear in any indicator or evidence quote.

---

## Rules Hypothesis Schema — v2

```json
{
  "version": "v2",
  "narratives_processed": 0,
  "consolidation_date": "...",
  "typology_rules": [
    {
      "typology": "MONEY_LAUNDERING",
      "fincen_tag": "821",
      "confidence": 0.0,
      "rules": [
        {
          "rule_id": "ML_821_001",
          "condition": "transaction.channel == 'P2P'",
          "threshold": "$X",
          "temporal_constraint": "N days",
          "indicators": [
            {
              "indicator_id": "FFI_F_FT_003",
              "indicator_text": "...",
              "source": "FFIEC_APPENDIX_F",
              "frequency": 6,
              "frequency_pct": 0.60
            }
          ],
          "supporting_case_count": 0,
          "supporting_totals": []
        }
      ]
    }
  ]
}
```

**Key schema changes from v1:**
- Indicators now carry indicator_id and source — traceable to vocabulary
- frequency and frequency_pct replace the implicit supporting_case_count per indicator
- threshold computed as min(supporting_totals) in Python — never set by LLM
- Channel derived from spaCy frames in Python — never set by LLM
- Rule IDs generated in Python — never set by LLM
- confidence = supporting_case_count / narratives_processed (unchanged)

---

## Intermediate Artifact Schema

### raw_indicator_matches.json

Append-only during Phase 1. Input to consolidate_hypothesis() in Phase 2.

```json
[
  {
    "case_id": "...",
    "narrative_date": "...",
    "active_fincen_tags": [
      {"fincen_tag": "821", "typology": "MONEY_LAUNDERING", "description": "..."}
    ],
    "suspicious_total": 0.0,
    "review_period": {"start": "...", "end": "..."},
    "spacy_frames": {
      "channels": [],
      "direction": "inflow",
      "amounts": [],
      "transaction_types": [],
      "red_flags": []
    },
    "matched_indicators": [
      {"indicator_id": "FFI_F_FT_003", "evidence_quote": "..."}
    ]
  }
]
```

---

## SAR Narrative Structure

Five formal sections, always present, always ALL CAPS headers
(exception: Suspicious Activity List):

1. INTRODUCTION — alert trigger, review period, accounts involved,
   total suspicious amount summary
2. DETAILS OF INVESTIGATION — customer profile, account summaries,
   transaction detail. Suspicious Activity List may be embedded here.
3. SUSPICIOUS ACTIVITY LIST — may appear as own section or embedded
   within Details of Investigation.
4. CONCLUSION — summary of findings, total suspicious amount confirmed
5. CONTENTS OF FILE — supporting documents list, not useful for extraction

LLM receives: Introduction + Conclusion as primary sections.
Details of Investigation as secondary — available but LLM instructed to
prioritize Introduction and Conclusion for quantitative signals.
SAR Monitoring Cases excluded (SARMON = 0 filter).

---

## FinCEN Code Mapping (unchanged from v1)

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

Key codes (current focus):
- 821 = Suspicious concerning the source of funds
- 824 = Funnel account
- 805 = Suspicious EFT/wire transfers
- 812 = Transaction out of pattern for customer(s)
- 807 = Suspicious use of multiple accounts
- 925 = Transaction with no apparent economic, business, or lawful purpose
- 911 = Two or more individuals working together

---

## File Structure — v2

```
/home/jupyter/custom_agent/typology_agent/
├── agent_tester.ipynb
├── cases_by_CASE_ID.ipynb
├── check_FINCEN_tag_tool.ipynb
├── check_SAR_tool.ipynb
├── config.yaml
├── GEMINI.md                          # This file (v2)
├── induct_tester.py                   # Standalone induction runner
├── main.py
├── parse_narrative_tester.ipynb
├── requirements.txt
├── srl_extract_tester.ipynb
├── evaluation/                        # NOT YET UPDATED for TCA
└── tca/
    ├── __init__.py
    ├── agent.py                       # TypologyClassificationAgent
    │                                  # induct() — v2 pipeline
    │                                  # consolidate_hypothesis() — NEW
    ├── app_config.py
    ├── llm_helpers.py                 # Single LLM call:
    │                                  # match_indicators() — REPLACES
    │                                  # extract_narrative_structure()
    │                                  # and synthesize_rules()
    ├── parse_narrative.py             # Unchanged from v1 ✅
    ├── sar_procedures.py              # Extended:
    │                                  # + HIGH_RISK_COUNTRIES constant
    │                                  # + load_ffiec_appendix_f()
    │                                  # + load_ffiec_appendix_g()
    │                                  # + get_indicator_vocabulary()
    ├── srl_extract.py                 # Unchanged from v1 ✅
    ├── typology_keywords.py           # Unchanged
    ├── data/
    │   ├── sar_procedure.json         # Existing ✅
    │   ├── ffiec_appendix_f.json      # NEW — to be built
    │   └── ffiec_appendix_g.json      # NEW — to be built
    ├── not used/                      # Legacy files, safe to ignore
    └── tools/
        ├── __init__.py
        ├── bq_tools.py
        ├── fincen_tag_tools.py        # get_fincen_tags() ✅
        ├── gcs_tools.py
        └── sar_narrative_tools.py     # get_sar_narratives() ✅
```

---

## Key Data Sources (unchanged from v1)

### SAR Narratives
- Function: get_sar_narratives(start_date, end_date, case_id=None)
- BQ Project: esa-src-prod-5736
- SAR Monitoring Cases excluded via: AND B.SARMON = 0
- Status: TESTED WORKING ✅

### FinCEN Tags
- Function: get_fincen_tags(start_date, end_date, case_id=None, target_fincen_tag=None)
- Status: TESTED WORKING ✅

### Config (config.yaml) — unchanged
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

## What's Working vs Pending — v2

### WORKING ✅
- get_sar_narratives() — SAR Monitoring Cases excluded
- get_fincen_tags() — target_fincen_tag filter working
- app_config.py + config.yaml
- parse_narrative.py — section splitter + suspicious_total + review_period
- srl_extract.py — amounts, channels, transaction_types, red_flags,
  account number filter, dates and persons removed
- sar_procedures.py — selective injection, HIGH_RISK_COUNTRIES constant
- tca/data/sar_procedure.json
- FINCEN_CODE_MAP + FINCEN_CODE_SECTION_REFINEMENT

### PENDING ⏳

**Immediate (next session):**
1. Build tca/data/ffiec_appendix_f.json from FFIEC Appendix F
   Source: https://bsaaml.ffiec.gov/manual/Appendices/07
   Each red flag → {indicator_id, category, text, source}
2. Build tca/data/ffiec_appendix_g.json from FFIEC Appendix G
   Source: https://bsaaml.ffiec.gov/manual/Appendices/08
3. Extend sar_procedures.py with load_ffiec_appendix_f(),
   load_ffiec_appendix_g(), get_indicator_vocabulary()
4. Rewrite llm_helpers.py: replace extract_narrative_structure() and
   synthesize_rules() with single match_indicators() call
5. Rewrite induct() in agent.py: v2 pipeline, Phase 1 only
   (per-narrative processing → raw_indicator_matches.json)
6. Build consolidate_hypothesis() in agent.py: Phase 2 post-corpus
   consolidation pass
7. Run induct() at limit=10, evaluate raw_indicator_matches.json quality
8. Run consolidate_hypothesis(), evaluate Rules Hypothesis v2

**Deferred:**
- Channel granularity review — AML vendor methodology comparison pending
  Current placeholder: BOT canonical channel values from spaCy frames
- validate_rules_hypothesis() — post-processing validator
- classify() — not yet built
- Braintrust — access not provisioned
- GitLab — access not sorted
- evaluation/configs/ — still CRA version
- Closed no-SAR cases / severity boundary agent — out of scope for TCA

---

## Immediate Next Steps (priority order)

1. Build ffiec_appendix_f.json and ffiec_appendix_g.json
2. Extend sar_procedures.py with vocabulary loading functions
3. Rewrite llm_helpers.py with match_indicators()
4. Rewrite induct() Phase 1 in agent.py
5. Build consolidate_hypothesis() Phase 2 in agent.py
6. Run at limit=10 and evaluate

---

## Key Design Decisions and Principles — v2

### LLM Scope is Matching, Not Synthesis
The LLM answers one question per narrative: which items from the controlled
indicator vocabulary are evidenced here? It does not synthesize rules, assemble
schemas, assign channels, compute thresholds, or generate rule IDs. All of those
are deterministic Python.

### Controlled Vocabulary Eliminates Hallucination at the Source
v1 asked the LLM to invent generalizable indicators freely. This produced bloat,
case-specific entity leakage, and cross-tag contamination regardless of prompt
constraints. v2 eliminates this by giving the LLM a fixed vocabulary to match
against. The LLM cannot invent an indicator that isn't in the vocabulary.

### FinCEN Tags as Output Validation, Not Input Filter
v1 used active_fincen_tags to scope LLM extraction per tag, causing per-tag
isolation calls and cross-contamination. v2 recognizes that SAR narratives contain
mixed evidence regardless of tag — the tag is the investigator's conclusion, not
a filter on evidence. Tags are used in provenance tracking and output validation
only.

### Two-Phase induct() Eliminates Mid-Stream Merging
v1 merged rules incrementally narrative-by-narrative, which required the LLM to
handle merge logic and produced inconsistent results. v2 separates extraction
(Phase 1, LLM involved) from consolidation (Phase 2, Python only). Phase 1 is
append-only. Phase 2 runs once after all narratives are processed.

### Threshold is Always Mathematical
threshold = min(supporting_totals) computed in Python from pre_parse
suspicious_total values. LLM never sets threshold. (Unchanged from v1 intent,
now actually enforced by architecture.)

### Channel Granularity is Deferred
Channel assignment uses BOT canonical values from spaCy frames as a placeholder.
Granularity review against AML vendor methodologies is pending before
consolidate_hypothesis() is finalized.

### Severity and Hidden Pattern Discovery is Out of Scope
TCA is guided extraction against a known vocabulary. Discovering severity
boundaries or hidden patterns requires comparison of filed vs. not-filed closed
cases — a separate future agent with different data requirements. TCA output
feeds ASM 2.0 as a candidate feature source; it does not itself determine severity.

### Agent 2 — BOT Package Agent (Future, unchanged)
Status: Deferred. Design only.
Condition field in Rules Hypothesis remains machine-parseable for Agent 2.
Only: transaction.channel, transaction.direction, transaction.amount.
BOT canonical channel values only in condition field.

---

## Tools and Infrastructure (unchanged)

- LLM: Google Gemini 2.5 Flash
- Infrastructure: GCP, Vertex AI, BigQuery, GCS
- Observability/Eval: Braintrust (stubbed)
- NLP: spaCy 3.8.14 + en_core_web_sm
- Libraries: google-adk, google-genai, spaCy, pandas
- Data: BigQuery (SAR narratives, FinCEN tags, MDM)
