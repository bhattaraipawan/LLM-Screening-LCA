# ELCD Catalog and Expert Reference Set

This directory contains the environmental-process catalog and expert-reference materials used by the controlled four-model LLM benchmark.

The purpose of this part of the repository is to create a traceable separation between:

```text
openLCA / ELCD database
        ↓
Fixed process catalog
        ↓
Independent human expert review
        ↓
Reconciled reference answers
        ↓
Frozen benchmark input
        ↓
LLM evaluation
```

The evaluated LLMs are not used to construct the expert reference set.

---

# 1. Directory Contents

The main files are:

```text
ELCD_Check/
├── ELCD_Process_Catalog.xlsx
├── README.md
└── expert_reference/
    └── LLM_LCA_Expert_Reference_Set_With_ELCD.xlsx
```

Related scripts are stored under:

```text
scripts/
├── export_openlca_process_catalog.py
└── prepare_benchmark_reference.py
```

The resulting frozen model-evaluation workbook is written to:

```text
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

---

# 2. ELCD Process Catalog

The controlled benchmark uses:

```text
ELCD_Check/ELCD_Process_Catalog.xlsx
```

The current exported catalog contains:

```text
608 processes
```

The catalog includes fields such as:

```text
process_uuid
process_name
category
location
library
process_type
```

Some metadata fields may be blank when the corresponding information is not returned in the exported openLCA process descriptor.

The core identifiers used by the benchmark are:

```text
process UUID
process name
available location
process type
```

---

# 3. Exporting the Catalog from openLCA

The catalog is exported using:

```text
scripts/export_openlca_process_catalog.py
```

The intended database for this study is:

```text
ELCD 3.2
```

Before exporting the catalog:

1. open openLCA;
2. activate the intended ELCD 3.2 database;
3. start the openLCA IPC server;
4. confirm that the configured IPC port is available; and
5. execute the export script.

Example:

```bash
python scripts/export_openlca_process_catalog.py
```

The preferred default output location is:

```text
ELCD_Check/ELCD_Process_Catalog.xlsx
```

---

# 4. Important Database-Provenance Note

The exporter reads processes from the database currently available through the openLCA IPC server.

The metadata field:

```text
database_label = ELCD 3.2
```

documents the intended study configuration.

It does **not** itself:

- switch the active openLCA database;
- independently identify the active database; or
- guarantee that ELCD 3.2 was selected.

Therefore database activation is an explicit prerequisite of the export procedure.

For reproducibility, the operator should confirm that ELCD 3.2 is active in openLCA before running the exporter.

---

# 5. Current Catalog Integrity

The current catalog contains:

```text
Total process records: 608
Blank process UUIDs: 0
Duplicate process UUIDs: 0
Blank process names: 0
```

The process catalog contains both LCI-result and unit-process descriptors.

The catalog file should remain unchanged during a formal model-comparison experiment.

---

# 6. Expert Reference Workbook

The expert reference workbook is:

```text
ELCD_Check/expert_reference/LLM_LCA_Expert_Reference_Set_With_ELCD.xlsx
```

It contains the principal sheets:

```text
Instructions
Expert_A
Expert_B
Reconciliation
ELCD_Catalog
QC_Summary
```

The embedded `ELCD_Catalog` sheet provides the process-selection list used by both human reviewers.

This prevents experts from manually entering arbitrary process names or UUIDs outside the benchmark catalog.

---

# 7. Embedded Catalog Consistency

The embedded expert-review catalog is intended to be an exact selection copy of:

```text
ELCD_Check/ELCD_Process_Catalog.xlsx
```

For the current reference workbook:

```text
External catalog UUIDs: 608
Embedded catalog UUIDs: 608
Missing UUIDs: 0
Extra UUIDs: 0
Process-name mismatches: 0
Location mismatches: 0
```

Therefore the experts and benchmark code use the same process universe.

---

# 8. Independent Expert Review Procedure

Two reviewers independently complete:

```text
Expert_A
Expert_B
```

Each reviewer receives the same:

- BOM material description;
- quantity;
- unit; and
- ELCD process catalog.

For every BOM item, the reviewer records:

1. normalized material name;
2. best available ELCD/openLCA process;
3. process UUID;
4. match type;
5. confidence; and
6. optional notes/rationale.

The reviewers should not see the evaluated model outputs before completing the reference set.

---

# 9. Process Selection

Experts select a process using the catalog selection label.

A selection label combines process-identification information in a readable form.

The corresponding process UUID is automatically filled from the embedded catalog.

This avoids manually copying UUIDs and reduces process-name ambiguity.

For Direct or Proxy classifications:

```text
Selected process required
Process UUID required
```

For Review Required:

```text
No selected ELCD process UUID
```

---

# 10. Match-Type Definitions

## Direct

Use Direct when a suitable ELCD process represents the original material/product sufficiently well for the study's screening purpose.

Examples can include a material where the catalog contains a directly corresponding generic process.

A Direct classification does not imply perfect geographic, temporal, or product-specific equivalence.

---

## Proxy

Use Proxy when:

- no sufficiently direct dataset is available; but
- an ELCD process can still serve as a technically defensible substitute.

Examples may include choosing a generic steel product dataset for a more specific small steel component when a component-specific ELCD process is absent.

Proxy should be used whenever an actual ELCD process is selected as a substitute.

---

## Review Required

Use Review Required only when no supplied ELCD process is sufficiently defensible for selection.

For a Review Required reference row:

```text
Final process UUID = blank
```

Review Required should not be used merely because:

- geography is imperfect;
- the process is generic;
- engineering judgment is required; or
- the process is not an exact product match.

If a substitute process is still selected, the appropriate class is Proxy.

---

# 11. Expert Confidence

The expert workbook supports:

```text
High
Medium
Low
```

confidence.

Confidence documents the reviewer's certainty in the selected interpretation/process.

It is not used as the benchmark target itself unless explicitly analyzed later.

---

# 12. Expert Agreement

For the current 35-item reference set, independent agreement before reconciliation was:

| Measure | Agreement |
|---|---:|
| Normalized material | 25/35 = 71.4% |
| Selected process | 23/35 = 65.7% |
| Match type | 29/35 = 82.9% |
| All three fields | 17/35 = 48.6% |

Rows with at least one expert disagreement:

```text
18/35
```

These disagreements are expected to be resolved before model scoring.

---

# 13. Reconciliation

The `Reconciliation` sheet combines the independent reviews.

The final benchmark fields are:

```text
Final Normalized Material
Final Reference Process
Final Process UUID
Final Decision
```

Where both experts agree, common values can be carried forward automatically.

Where they disagree, the final answer is resolved through expert reconciliation.

The final reference must not be determined using model performance.

---

# 14. Recommended Reconciliation Documentation

For disagreement rows, a short reconciliation note is recommended.

The note should document why the final:

- normalization;
- process; and/or
- match type

was selected.

This is particularly useful when:

- the experts selected different proxy processes;
- one expert selected Direct and the other Proxy;
- the final normalized term differs from both initial terms; or
- all major fields required reconciliation.

Reconciliation notes should reflect the actual human decision and should not be invented after model results are observed.

---

# 15. Final Reference Distribution

The current reconciled reference contains:

```text
Direct          13
Proxy           15
Review Required  7
Total           35
```

All final Direct/Proxy UUIDs must exist in the same exported ELCD catalog.

All final Review Required rows must contain no selected UUID.

---

# 16. Quality-Control Summary

The workbook contains a:

```text
QC_Summary
```

sheet.

It summarizes:

- expert agreement;
- disagreement fields;
- final normalized material;
- final reference process;
- final process UUID;
- final match decision; and
- final completion status.

The reference should be treated as ready for model evaluation only after all 35 rows have complete final labels.

---

# 17. Freezing the Reference Set

After reconciliation, run:

```bash
python scripts/prepare_benchmark_reference.py
```

The script reads the expert workbook and creates:

```text
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

The script is intended to create a clean model-scoring workbook rather than using the entire expert-review workbook directly.

---

# 18. Validation Performed During Freezing

The preparation workflow verifies items such as:

- expected benchmark rows are present;
- sample IDs are unique;
- original BOM descriptions remain consistent;
- final normalized material is present;
- final match type is valid;
- Direct/Proxy rows contain a catalog process;
- process UUIDs exist in the exported catalog;
- selected process names correspond to catalog UUIDs;
- Review Required rows do not contain selected process UUIDs; and
- final labels are complete.

If the reference is unfinished or inconsistent, the benchmark should not proceed.

---

# 19. Frozen Reference

The output workbook is:

```text
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

This file contains the final scoring labels used by:

```text
scripts/benchmark_four_llms.py
```

Once formal model evaluation begins, the frozen reference should not be edited.

If expert labels must genuinely change, the complete benchmark should be rerun using the new frozen reference.

---

# 20. Validate the Frozen Reference

Run:

```bash
python scripts/benchmark_four_llms.py --check-inputs
```

For the current reference set, the expected high-level output is:

```text
Catalog processes: 608

Reference rows: 35
Matched Direct/Proxy rows: 28
Review Required rows: 7

Deterministic Top-5 candidate-pool recall:
21/28 = 75.0%

TF-IDF Top-1 baseline:
11/28 = 39.3%
```

---

# 21. Relationship to the LLM Benchmark

The ELCD catalog and expert workbook provide the ground truth for the controlled evaluation.

The benchmark workflow is:

```text
Original BOM material
        ↓
Character n-gram TF-IDF
        ↓
Five candidate ELCD processes
        ↓
Deterministic shuffle
        ↓
LLM
        ↓
Normalized material
+
Ranked candidate UUIDs
+
Direct / Proxy / Review Required
        ↓
Comparison with frozen expert reference
```

The expert answers are used only for evaluation, not to create the TF-IDF query or candidate pool.

---

# 22. Candidate Retrieval

The retriever uses:

```text
original BOM description only
```

with:

```text
character n-gram TF-IDF
3–5 character n-grams
Top-5 candidate pool
```

The following expert information is not supplied to the retriever:

```text
final normalized material
final reference process
final process UUID
final match type
```

This prevents reference leakage.

---

# 23. Candidate-Pool Performance

Among the 28 final Direct/Proxy reference rows:

```text
21 expert processes occur in Top-5
```

therefore:

```text
Top-5 retrieval recall = 75.0%
```

The first-ranked TF-IDF process equals the expert reference for:

```text
11/28 = 39.3%
```

This provides an independent lexical baseline.

---

# 24. Why the Candidate Pool Is Five

A diagnostic test using 20 candidates produced the same expert-process coverage:

```text
Top-5  = 21/28
Top-20 = 21/28
```

Increasing the candidate pool therefore did not recover any additional expert-reference process.

The formal benchmark retains five candidates because this:

- minimizes unnecessary prompt length;
- limits distractors;
- retains the same measured reference coverage; and
- produces a clearer candidate-selection task.

---

# 25. Why Retrieval Scores Are Hidden

The LLM should evaluate process meaning rather than simply follow the lexical retriever.

Therefore the benchmark does not expose:

```text
TF-IDF similarity score
TF-IDF original rank
```

to the model.

The five retrieved candidates are instead presented in a deterministic shuffled order that is common across the evaluated models.

The original retrieval ranking is retained internally only for retrieval evaluation and the TF-IDF baseline.

---

# 26. Scope of the Expert Reference

The expert reference evaluates process matching within the fixed exported ELCD catalog.

It does not establish that every selected process is:

- product-specific;
- geographically ideal;
- temporally ideal;
- equivalent to a manufacturer EPD; or
- universally preferable for all LCA applications.

Proxy classification explicitly acknowledges database limitations.

The reference should therefore be interpreted as an expert benchmark for the defined screening workflow and fixed catalog.

---

# 27. Geographic Considerations

The catalog contains processes from different geographic scopes, including examples such as:

```text
RER
EU-27
DE
GLO
```

The study evaluates whether a process is sufficiently defensible for the screening/matching task.

Geographic representativeness should be documented separately as a limitation when appropriate.

---

# 28. Important Experimental Rules

Before formal benchmarking:

```text
Expert reconciliation must be complete.
```

During formal benchmarking, do not alter:

```text
ELCD_Check/ELCD_Process_Catalog.xlsx
ELCD_Check/expert_reference/LLM_LCA_Expert_Reference_Set_With_ELCD.xlsx
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

after model inference has begun.

Do not adjust the expert reference after viewing which answers favor a particular LLM.

Do not add post-hoc reference-derived synonyms merely to improve retrieval performance.

If the underlying expert reference is legitimately revised, regenerate the frozen benchmark and rerun all models consistently.

---

# 29. Reproducing the Complete Data Chain

The intended sequence is:

## Step 1 — Activate ELCD in openLCA

Open:

```text
ELCD 3.2
```

and start the IPC server.

## Step 2 — Export catalog

```bash
python scripts/export_openlca_process_catalog.py
```

Expected output:

```text
ELCD_Check/ELCD_Process_Catalog.xlsx
```

## Step 3 — Complete independent expert review

Use:

```text
ELCD_Check/expert_reference/LLM_LCA_Expert_Reference_Set_With_ELCD.xlsx
```

## Step 4 — Reconcile expert answers

Complete all final fields in:

```text
Reconciliation
```

## Step 5 — Freeze benchmark reference

```bash
python scripts/prepare_benchmark_reference.py
```

## Step 6 — Validate

```bash
python scripts/benchmark_four_llms.py --check-inputs
```

## Step 7 — Smoke test

```bash
python scripts/run_four_llm_benchmark.py --smoke
```

## Step 8 — Formal benchmark

```bash
python scripts/run_four_llm_benchmark.py
```

---

# 30. Summary

This directory provides the fixed environmental-process universe and independent human reference needed to evaluate the four LLMs fairly.

The key methodological principle is:

```text
The expert reference is created first.
The candidate retriever is deterministic.
The model sees only the original material and supplied candidates.
The model cannot invent environmental-process records.
The model output is evaluated against the frozen human reference.
```

This separation is necessary for a controlled and reproducible evaluation of LLM-assisted construction-material interpretation and ELCD process matching.
