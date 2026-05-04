# TCA Applications for ASM Enhancement

**Financial Crimes Risk Management | KeyBank**
*Brainstorm Summary — May 2026*

---

## Overview

This document summarizes two analytical applications of the Typology Classification Agent (TCA) that are intended to support Alert Scoring Model (ASM) enhancement. The TCA is a GCP-based LLM pipeline built on Vertex AI and Gemini 2.5 Flash. Its core capability is reading SAR narratives against formalized AML procedure — specifically FinCEN Appendix F and KeyBank's internal SAR procedure document — and producing a structured, per-narrative indicator profile grounded in that procedure. The two functions that implement this are `induct()`, which builds a typology knowledge base from filed SAR narratives, and `classify()`, which applies that knowledge base to score unseen narratives against inducted indicator profiles.

The applications described below are diagnostic and directional in nature. The TCA does not engineer transaction features, does not compute anything from raw transaction data, and does not directly modify the ASM model. Its contribution is surfacing investigator-validated behavioral intelligence that the transaction-based feature space cannot produce on its own, and directing the ASM feature ideation process toward specific, evidence-grounded gaps.

---

## Application 1: Boundary SAR Characterization for Red-Flag Feature Ideation

### Background

ASM has a current auto-clear threshold of 30% and a target threshold of 50%. The correlations scoring between 30% and 50% represent the population that ASM cannot confidently resolve — cases that fall above the current clear threshold but below where the model would need to be to support the expanded auto-clear. Within this boundary population, correlations that resulted in SAR filings (SAR=1) are analytically accessible because they have SAR narratives and FinCEN tags recorded in BigQuery. A separate reference population also exists: high-confidence SAR=1 cases scoring above 50%, where ASM is already scoring confirmed suspicious behavior correctly.

### What the TCA Does

`classify()` is run against both populations: the boundary SAR=1 narratives (30%–50%) and the high-confidence SAR=1 narratives (>50%). For each narrative, `classify()` returns a FinCEN tag classification and a matched indicator profile drawn from the `rules_hypothesis` vocabulary, which is grounded in Appendix F and SAR procedure. Indicators are the behavioral descriptors extracted from narratives and matched programmatically via cosine similarity to the inducted vocabulary — they represent the qualitative dimensions of suspicious behavior that investigators observed and described at the time of filing.

### The Analysis

Results are grouped by FinCEN tag first, then indicator frequency distributions are compared between the two populations within each tag group. Indicators that appear consistently in boundary SAR=1 cases but are relatively absent or weaker in the high-confidence SAR=1 reference represent behavioral signals that are present in confirmed suspicious activity but that ASM is currently discounting or not capturing strongly enough to push the correlation score above 50%.

### The Output

The output is a ranked, typology-grouped list of indicators associated with ASM's blind spots for confirmed suspicious behavior near the decision boundary. This list is reviewed by a data scientist who assesses whether each indicator has a transaction-based analog — a behavioral pattern that can be computed from Cash, Money Instrument, or Wire transaction tables on GCP. Indicators that are feasibly engineerable from transaction data become candidate red-flag feature definitions and enter the standard ASM feature engineering pipeline. Indicators that exist only in narrative text and have no transaction analog are documented as structural limitations of a transaction-based model.

### Scope and Limitations

This application is limited to the ATL SAR=1 population. Correlations that were not filed (SAR=0) do not have narratives and are out of scope. The analysis is one-sided by design — it characterizes where ASM underweights confirmed suspicious behavior, not where it over-scores. The TCA does not produce new feature values; it produces candidate feature definitions that require downstream engineering validation before they can be incorporated into ASM.

---

## Application 2: Red-Flag Coverage Gap Detection via Unmapped Indicator Residual

### Background

ASM's feature set includes a red-flag feature layer consisting of approximately 1,639 candidate features. Each red-flag feature is defined around a specific suspicious behavior — a behavior type, a suspicious period (e.g. 1, 2, 4, 6, or 9 days), a lookback window (currently 3 months for red-flag features), and a set of metrics. These definitions were developed by SMEs based on prior experience and investigator intuition and represent the current state of typology-aware behavioral coverage in the model.

The TCA's indicator vocabulary is built by a different method. `induct()` processes filed SAR narratives grounded in Appendix F and SAR procedure and derives, empirically and from the bottom up, the behavioral patterns that investigators consistently describe when filing. The two vocabularies — SME-designed red-flag features and TCA-inducted indicators — are complementary but were constructed independently. There is no guarantee they cover the same behavioral space.

### What the TCA Does

`classify()` is run across a broad corpus of SAR narratives spanning all inducted typologies. For each narrative, the full set of matched indicators is collected. Aggregated across the corpus, this produces a population-level indicator frequency distribution — a ranked view of which behavioral patterns appear most consistently across confirmed SAR filings.

### The Analysis

Each TCA indicator in the frequency distribution is mapped against ASM's existing red-flag behavioral dimension taxonomy. This mapping is qualitative — dimension to dimension, not numeric threshold to numeric threshold. The reason is that SAR narratives are sparse on exact quantitative detail; the TCA extracts behavioral patterns, not transaction counts or amounts. The mapping produces two buckets: indicators that correspond to an existing ASM red-flag behavioral dimension, confirming that ASM already has a feature designed to detect that behavior; and indicators that have no corresponding red-flag dimension in ASM's current feature set. The second bucket is the residual.

### The Output

The unmapped residual is a prioritized list of behavioral gap candidates — patterns that investigators consistently describe in filed SARs that ASM has no feature designed to detect. Each candidate is reviewed by a data scientist for engineering feasibility: can this behavioral pattern be computed from the transaction tables available on GCP? Feasible candidates enter the standard ASM red-flag feature engineering pipeline. Infeasible candidates — behaviors that are only observable in investigator narrative and have no transaction-level analog — are documented as a structural boundary of a transaction-based model.

### Scope and Limitations

This application functions as a systematic, empirically grounded coverage audit of ASM's red-flag feature layer. It does not replace SME-driven feature design; it supplements it with a bottom-up view of what investigators actually describe when filing, which may surface patterns that top-down SME design did not anticipate. The mapping step between TCA indicators and ASM red-flag dimensions requires human judgment and is not automated. The output is a prioritized candidate list, not a validated feature set — engineering feasibility and model impact require separate validation through the standard ASM feature development process.

---

## Summary

Both applications use the TCA as a reading and classification tool grounded in formalized AML procedure. Neither requires the TCA to produce quantitative transaction output, which is consistent with what SAR narrative text can reliably support. Both produce outputs that are interpretable by a data scientist and actionable within the existing ASM feature engineering pipeline.

Application 1 is targeted — it focuses diagnostic effort on the specific boundary population where ASM improvement is needed most. Application 2 is broader — it audits ASM's entire red-flag coverage posture against empirical evidence from investigator filings. Together they provide a narrative-grounded, procedure-backed basis for ASM feature ideation that is complementary to, and not duplicative of, the transaction-based analysis the ASM development team already performs.
