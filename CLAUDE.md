# Typology Classification Agent (TCA) — Project Context

## Project Overview

The Typology Classification Agent (TCA) is a GCP/Vertex AI-based AI agent being developed
at KeyBank within the Financial Crimes Risk Management team. It is Phase 1 of a two-phase
AML multi-agent SAR automation system. The agent's purpose is to learn to classify AML
typologies from historical SAR narratives with ground truth FinCEN tags, and then apply
that learned Rules Hypothesis to classify unseen SAR narratives.

Phase 2 (not yet started) is the Narrative Reasoning Agent, which takes the TCA's output
and generates full SAR narratives for new cases using BOT packages as primary input.

---

## Business Context

- **Team**: Financial Crimes Risk Management, KeyBank
- **Environment**: Vertex AI Workbench (GCP), VS Code with Gemini Code Assist
- **Primary language**: Python
- **LLM**: Gemini 2.0 Flash via Google ADK (google-adk)
- **Orchestration**: Google ADK (Agent Development Kit) — refactor in progress
- **Data warehouse**: BigQuery (project: esa-src-prod-5736)
- **Observability**: Braintrust (tracing, experiment tracking, prompt versioning)
- **Proxy**: vzproxy.keybank.com (required for all pip installs)

---

## Architecture Overview

The TCA pipeline has three run modes driven from main.py:

### Mode 1: induct
1. Pull historical SAR narratives from BigQuery (sar_narrative_tools.py)
2. Pull FinCEN typology tags from BigQuery (fincen_tag_tools.py)
3. Join both datasets on ITEM_ID — this produces the labeled training corpus
4. Generate a formalized Reading Plan via Gemini (llm_helpers.py)
5. For each narrative, run two sequential Gemini calls:
   - Call 1: extract_narrative_structure() — parse SAR into schema-enforced
     behavioral components
   - Call 2: synthesize_rules() — iteratively accumulate Rules Hypothesis
6. Output: Rules Hypothesis (structured JSON artifact) + Reading Plan

### Mode 2: classify
1. Load Rules Hypothesis (from induct or persisted store)
2. Take unseen SAR narrative as input
3. Run classify_narrative() via Gemini
4. Output: ranked typology candidates with confidence scores
   e.g. [Structuring 0.87, Layering 0.61, Money Laundering 0.45]
5. Output routed to either:
   - Braintrust (eval mode — automated scoring vs held-out FinCEN tags)
   - HITL/SME review (production mode — human confirms or corrects)

### Mode 3: eval
1. Pull held-out SAR narratives + FinCEN tags (date range separate from induction)
2. Run classify() against each case
3. Log expected vs actual to Braintrust for quantitative scoring
4. SME qualitative review follows (per section 2.4 of spec)

---

## Spec Reference (sections 2.1-2.4)

### 2.1 Authentication, Configuration, and Data Ingestion
- Three inputs: BOT packages (Excel), FinCEN tags, historical SAR narratives
- All hosted on GCP (Cloud Storage + BigQuery)
- GCP service accounts with IAM roles for read-only access
- DLP for PII de-identification (planned, not yet implemented)
- Vertex AI RAG Engine for retrieval (planned)
- Document AI for unstructured doc parsing (planned)

### 2.2 Narrative Ingestion to Reading Plan and Narrative Structure
- Reading Plan: formalized structured prompt defining how agent reads SAR narratives
- Generated via RAG over historical SAR narratives
- Narrative Structure: formalized prompt defining how agent writes SAR narratives
  (used in Phase 2 Narrative Reasoning Agent)
- SME feedback loop on both outputs

### 2.3 Rule Hypothesis Generation
- Two sequential Gemini API calls:
  - Call 1: Structured narrative extraction → schema-enforced behavioral components
  - Call 2: Inductive rule synthesis → accumulates rules across narrative population
- Candidate rule store maintained as vector index (Vertex AI Vector Search — planned)
- Output: Rules Hypothesis artifact with provenance metadata

### 2.4 Typology Identification Testing and Success Criteria
- Quantitative: Gemini evaluates held-out narratives against Rules Hypothesis
- Qualitative: AML SME review of induced rules
- Minimum coverage threshold defined with internal stakeholders
- Both stages must pass for Rules Hypothesis to be promoted downstream

---

## File Structure
