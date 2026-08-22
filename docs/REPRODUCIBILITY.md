# Reproducibility Guide

This document describes how to reproduce the controlled four-model benchmark for LLM-assisted material normalization and ELCD/openLCA process matching.

The experiment is designed to separate:

1. expert-reference construction;
2. deterministic candidate retrieval;
3. LLM interpretation and process selection;
4. Review Required routing; and
5. repeatability/stability.

The benchmark does not ask the evaluated LLMs to generate environmental impact factors or GWP values.

---

# 1. Experimental Overview

The controlled experiment evaluates four open-weight instruction-tuned LLMs:

| Model | Checkpoint |
|---|---|
| Llama 3.1 8B Instruct | `meta-llama/Llama-3.1-8B-Instruct` |
| Qwen2.5 7B Instruct | `Qwen/Qwen2.5-7B-Instruct` |
| DeepSeek LLM 7B Chat | `deepseek-ai/deepseek-llm-7b-chat` |
| Mistral 7B Instruct v0.3 | `mistralai/Mistral-7B-Instruct-v0.3` |

All models are evaluated using the same:

- 35-item frozen expert reference;
- ELCD process catalog;
- deterministic candidate-retrieval procedure;
- five-process candidate pool;
- candidate presentation order;
- task instructions;
- maximum output length;
- deterministic decoding configuration; and
- scoring logic.

---

# 2. Reference Dataset

The final expert reference contains:

```text
35 materials
```

distributed as:

```text
Direct          13
Proxy           15
Review Required  7
```

The source expert workbook is:

```text
ELCD_Check/expert_reference/LLM_LCA_Expert_Reference_Set_With_ELCD.xlsx
```

The machine-readable frozen benchmark is:

```text
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

The frozen benchmark should be generated only after expert review and reconciliation are complete.

---

# 3. Independent Expert Review

Two reviewers independently evaluate each BOM item.

For every item, the reviewer records:

- normalized material;
- preferred ELCD/openLCA process;
- automatically resolved process UUID;
- match type;
- confidence; and
- optional rationale.

Allowed match types are:

## Direct

A suitable ELCD process represents the material/product sufficiently well.

## Proxy

An exact/direct dataset is unavailable, but a technically defensible substitute is available.

## Review Required

No usable ELCD process is available.

Review Required rows must not contain a final process UUID.

The evaluated LLMs are not used to create the expert ground truth.

---

# 4. Expert Agreement

Before reconciliation, agreement for the 35 materials was:

```text
Normalized-material agreement:
25/35 = 71.4%

Selected-process agreement:
23/35 = 65.7%

Match-type agreement:
29/35 = 82.9%

Full agreement:
17/35 = 48.6%

Rows with at least one disagreement:
18/35
```

Disagreements are resolved before model evaluation.

The reconciled labels are treated as frozen reference answers during model scoring.

Reference labels must not be modified in response to model performance.

---

# 5. ELCD Catalog

The process catalog is stored at:

```text
ELCD_Check/ELCD_Process_Catalog.xlsx
```

The current catalog contains:

```text
608 processes
```

The catalog is exported from the database active in openLCA using:

```text
scripts/export_openlca_process_catalog.py
```

The intended study database is ELCD 3.2.

Before exporting:

1. launch openLCA;
2. activate the intended ELCD 3.2 database;
3. start the openLCA IPC server;
4. confirm the configured IPC port; and
5. run the exporter.

The exporter reads process descriptors from the database active through the IPC connection.

The stored `database_label` is study provenance metadata. It does not independently switch or verify the active openLCA database.

---

# 6. Catalog Integrity

The current exported catalog contains:

```text
608 process rows
0 blank process UUIDs
0 duplicate process UUIDs
0 blank process names
```

The expert workbook contains an embedded selection copy of the same catalog.

The embedded expert catalog and external benchmark catalog should contain the same:

- process UUIDs;
- process names; and
- location information.

This prevents the human reviewers from selecting processes from a different catalog than the one used during model scoring.

---

# 7. Freeze the Expert Reference

Run:

```bash
python scripts/prepare_benchmark_reference.py
```

This reads the reconciled human workbook and writes:

```text
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

The preparation script validates that:

- sample IDs are present and unique;
- required final normalized materials are present;
- match types are valid;
- Direct/Proxy rows resolve to an exact catalog UUID;
- the UUID exists in the exported catalog;
- process names correspond to their UUIDs;
- Review Required rows do not contain final process UUIDs; and
- the final reference is suitable for scoring.

The reference should then remain unchanged for the formal model experiment.

---

# 8. Validate Inputs

Before loading any LLM, run:

```bash
python scripts/benchmark_four_llms.py --check-inputs
```

For the current benchmark, expected values are:

```text
Catalog processes: 608

Reference rows: 35
Matched Direct/Proxy rows: 28
Review Required rows: 7

Top-5 candidate-pool recall:
21/28 = 75.0%

TF-IDF Top-1 baseline:
11/28 = 39.3%
```

This validation step does not require model inference.

---

# 9. Candidate Retrieval

Candidate retrieval is deterministic and independent of the evaluated LLM.

The retriever uses:

```text
TfidfVectorizer
analyzer = char_wb
ngram_range = (3, 5)
sublinear_tf = True
norm = l2
```

The retrieval query is constructed from:

```text
original BOM description only
```

The following are not used to retrieve candidates:

- expert normalized material;
- expert process name;
- expert process UUID;
- final match type;
- model output.

This protects the benchmark from ground-truth leakage.

---

# 10. Candidate Pool

The formal candidate-pool size is:

```text
5
```

For the 28 Direct/Proxy reference rows:

```text
21/28 = 75.0%
```

of expert-reference processes are recovered within the Top-5 candidate pool.

A diagnostic increase to 20 candidates did not improve this value.

Therefore the formal benchmark retains Top-5 because additional candidates did not improve expert-reference coverage and would increase prompt length and distractor count.

---

# 11. TF-IDF Baseline

TF-IDF itself provides a deterministic process-ranking baseline.

The expert process is the first-ranked TF-IDF candidate in:

```text
11/28 = 39.3%
```

of matched Direct/Proxy materials.

This value is reported independently of LLM performance.

It provides a baseline for evaluating whether an LLM adds value beyond simply accepting the lexical retriever's first-ranked process.

---

# 12. Candidate Presentation

The TF-IDF retrieval score and original TF-IDF ranking are not shown to the evaluated LLM.

The retriever first determines the Top-5 set.

The candidate set is then presented to the LLM in a deterministic shuffled order.

The same shuffled candidate ordering is used for all four models for a given material.

Conceptually:

```text
608 ELCD processes
       ↓
Character n-gram TF-IDF
       ↓
Top-5 candidate SET
       ↓
Store original TF-IDF ranking for baseline evaluation
       ↓
Deterministic candidate shuffle
       ↓
LLM receives five process options
```

This design reduces the possibility that a model simply follows the lexical retriever's rank.

---

# 13. Information Supplied to the LLM

The model receives:

- sample identifier;
- original material description;
- available quantity;
- available unit;
- five candidate process UUIDs;
- process names;
- available location information;
- process type; and
- task instructions.

The model does not receive:

- the expert normalized material;
- expert reference process;
- expert process UUID;
- expert match type;
- candidate retrieval scores; or
- original TF-IDF rank.

---

# 14. LLM Task

For each material, the model must:

1. normalize the construction material description;
2. rank up to three of the supplied candidate process UUIDs; and
3. classify the result as:
   - Direct,
   - Proxy, or
   - Review Required.

The model is restricted to the supplied process candidates.

The controlled benchmark explicitly prohibits generation of:

- new UUIDs;
- emission factors;
- GWP values;
- EPD values;
- citations; or
- invented environmental records.

---

# 15. Inference Configuration

The formal inference configuration is:

```text
Temperature:             0.0
Sampling:                False
Decoding:                Greedy
Candidate pool:          5
Reported LLM ranking:    Top 3
Maximum new tokens:      256
Quantization:            4-bit NF4
Main seed:               42
```

The same principal settings are used across the four models.

---

# 16. Main Benchmark

The main evaluation contains:

```text
35 materials
×
1 inference per model
×
4 models
=
140 responses
```

This provides the primary model-comparison results.

---

# 17. Repeatability Evaluation

A fixed 12-item subset is used for one additional inference pass.

The subset is balanced:

```text
4 Direct
4 Proxy
4 Review Required
```

The repeatability materials are fixed in advance rather than selected after seeing model results.

The additional experiment contains:

```text
12 materials
×
1 additional inference
×
4 models
=
48 additional responses
```

Therefore:

```text
140 main responses
+
48 repeatability responses
=
188 formal responses
```

---

# 18. Interpretation of Repeatability

Because the main benchmark uses greedy decoding with temperature 0.0, the repeatability experiment is primarily a test of deterministic/test-retest stability.

It should not be interpreted as a comprehensive assessment of stochastic variability.

Repeatability reporting must include valid-output coverage.

A repeated invalid output is not considered successful agreement merely because the same invalid output occurred twice.

Relevant repeatability measures include:

- both-runs-valid coverage;
- normalization agreement among valid pairs;
- selected-process/Review Required agreement among valid pairs;
- match-type agreement among valid pairs; and
- strict overall agreement requiring valid responses in both runs.

---

# 19. Smoke Test

Before running the formal experiment, use:

```bash
python scripts/run_four_llm_benchmark.py --smoke
```

The smoke test evaluates a small number of materials for each model.

Smoke-test results are stored separately under:

```text
Four_Models/Output/smoke/
```

and should not be mixed with formal benchmark workbooks.

The smoke test should be inspected for:

- successful model loading;
- valid structured outputs;
- generation errors;
- JSON parsing failures; and
- token-limit truncation.

---

# 20. Formal Four-Model Run

Run:

```bash
python scripts/run_four_llm_benchmark.py
```

The runner executes the four models in separate subprocesses.

This is particularly useful in Google Colab because each subprocess terminates after a model finishes, allowing GPU memory to be released before the next checkpoint is loaded.

The intended sequence is:

```text
Prepare/freeze reference
        ↓
Validate benchmark inputs
        ↓
Llama
        ↓
Qwen
        ↓
DeepSeek
        ↓
Mistral
        ↓
Repeatability evaluation
        ↓
Combined workbook
```

Do not modify the benchmark scripts, frozen reference, or ELCD catalog while a formal multi-model run is in progress.

---

# 21. Output Structure

Formal results are stored under:

```text
Four_Models/Output/
```

Expected model files:

```text
Four_Models/Output/llama/benchmark_results.xlsx
Four_Models/Output/qwen/benchmark_results.xlsx
Four_Models/Output/deepseek/benchmark_results.xlsx
Four_Models/Output/mistral/benchmark_results.xlsx
```

Combined comparison:

```text
Four_Models/Output/combined/four_model_comparison.xlsx
```

Repeatability:

```text
Four_Models/Output/repeatability/
```

Smoke results:

```text
Four_Models/Output/smoke/
```

---

# 22. Model Workbook Contents

Each formal model workbook contains sheets for items such as:

```text
Predictions
Metrics
Metadata
Prompt
```

The Predictions sheet contains the original BOM item, supplied candidate information, model output, reference labels, and derived scoring fields.

The Metrics sheet summarizes model performance.

The Metadata sheet records experimental configuration and available software/hardware provenance.

The Prompt sheet records the benchmark instructions/template.

---

# 23. Reproducibility Metadata

The workbooks record available fields such as:

```text
script version
database label
model key
model display name
model checkpoint
model revision
tokenizer revision
seed
candidate pool size
retrieval method
retrieval analyzer
retrieval n-gram range
retrieval query source
candidate presentation method
reported Top-k
max new tokens
temperature
decoding mode
quantization
Python version
platform
GPU
CUDA version
PyTorch version
Transformers version
Accelerate version
BitsAndBytes version
catalog path
catalog hash
reference path
reference hash
prompt hash
```

Content hashes help determine whether models were evaluated against the same reference and process catalog.

---

# 24. Structured-Output Reliability

Model responses are parsed and validated.

A response is considered valid only if it satisfies the required output constraints.

Potential failures include:

- empty response;
- malformed JSON;
- invalid match type;
- process UUID outside the supplied candidate set;
- selected process absent from the model ranking;
- Review Required combined with an invalid selected process; or
- generation failure.

Invalid responses are treated as failures during scoring rather than silently excluded from overall model performance.

---

# 25. Token-Limit Monitoring

The maximum generation length is:

```text
256 new tokens
```

Generated-token counts and token-limit information are retained so that truncation can be distinguished from genuine reasoning/selection errors.

This is particularly important for fair comparison between models with different response-generation behavior.

---

# 26. Evaluation Metrics

## 26.1 Normalization

Reported using:

- exact normalized-material accuracy;
- normalized-text similarity.

## 26.2 Candidate retrieval

Reported independently of the LLM:

- Top-5 candidate-pool recall;
- TF-IDF Top-1 baseline.

## 26.3 Candidate ranking

For matched reference rows:

- Top-1 ranking accuracy;
- Top-3 ranking recall;
- reciprocal rank;
- mean reciprocal rank.

## 26.4 Final process selection

Two versions should be reported.

### Overall process-selection accuracy

```text
Correct selected expert UUID
/
all 28 matched Direct/Proxy reference rows
```

This reflects the complete retrieval + model-selection pipeline.

### Conditional process-selection accuracy

```text
Correct selected expert UUID
/
matched cases where the expert UUID was actually present in Top-5
```

This evaluates LLM selection after successful retrieval.

The denominator for the current frozen benchmark is therefore based on the 21 successful retrieval cases.

## 26.5 Match classification

Reported using:

- overall match-type accuracy;
- matched-row Direct/Proxy accuracy;
- Review Required accuracy;
- Review Required precision;
- Review Required recall;
- Review Required F1.

## 26.6 End-to-end performance

A row is counted as end-to-end correct when:

- the selected expert process is correct for a Direct/Proxy reference row; or
- Review Required is correctly identified for an unmatched reference row.

---

# 27. Why Retrieval and Selection Are Reported Separately

If the reference process is absent from the candidate pool, the LLM cannot select that exact UUID.

For example:

```text
Reference process absent from Top-5
        ↓
LLM receives five different processes
        ↓
Exact expert UUID cannot be selected
```

Counting this only as an LLM error would confound retrieval failure with selection failure.

Therefore the study reports:

```text
Retrieval performance
+
Conditional LLM selection performance
+
Overall pipeline performance
```

separately.

---

# 28. Retrieval Limitation

The current lexical retrieval method achieves:

```text
Top-5 recall = 75.0%
```

for the matched reference rows.

Increasing the pool to 20 did not improve recovery.

The missing reference relationships primarily involve semantic proxy mappings where the BOM term and selected generic database process have limited lexical similarity.

No reference-derived synonym dictionary is added after observing these results because doing so could introduce target leakage.

More advanced semantic retrieval may be investigated separately in future work.

---

# 29. ELCD Geographic Limitations

The exported ELCD catalog includes processes with various geographic scopes such as:

```text
RER
EU-27
DE
GLO
```

The expert reference therefore evaluates whether a process is sufficiently defensible for the screening application, rather than asserting that every selected dataset is geographically representative of the case-study location.

Geographic representativeness is a separate source of uncertainty and should not be confused with language-model matching accuracy.

---

# 30. Recommended Experimental Record

For a publication-quality experiment, retain:

1. the exact repository commit used for inference;
2. the frozen reference workbook;
3. the exact ELCD catalog;
4. the four main result workbooks;
5. the four repeatability result workbooks;
6. the combined comparison workbook;
7. Colab/GPU information;
8. exact model revisions where available; and
9. the final manuscript prompt/configuration description.

Do not regenerate or edit reference labels after reviewing model performance.

---

# 31. Recommended Paper Description

A concise methodological description is:

> Four open-weight instruction-tuned LLMs were evaluated using a frozen 35-item expert reference set. A deterministic character n-gram TF-IDF retriever generated a common Top-5 ELCD candidate set for each material using only the original BOM description. The retriever's similarity scores and ranking positions were withheld from the models; instead, candidates were presented in a deterministic shuffled order common to all models. Each LLM normalized the material description, ranked up to three supplied process UUIDs, and classified the result as Direct, Proxy, or Review Required. Greedy decoding at temperature 0.0 and 4-bit NF4 quantization were used. Candidate retrieval, conditional model selection, overall process selection, match classification, structured-output reliability, inference time, and repeatability were evaluated separately.

---

# 32. Recommended Retrieval Results Description

For the current frozen reference:

> The deterministic retrieval stage recovered the reconciled expert process within the Top-5 candidate set for 21 of 28 matched materials (75.0%). The reference process was ranked first by TF-IDF in 11 of 28 matched cases (39.3%), providing a deterministic lexical-selection baseline. Because retrieval failures prevent downstream models from selecting the exact reference UUID, overall and conditional process-selection accuracies were reported separately.

---

# 33. Important Experimental Rule

Once formal inference begins, do not modify:

```text
scripts/benchmark_four_llms.py
scripts/run_four_llm_benchmark.py
scripts/prepare_benchmark_reference.py
ELCD_Check/ELCD_Process_Catalog.xlsx
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

until all four models and repeatability runs are complete.

This ensures all models are evaluated under the same experimental state.
