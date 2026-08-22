# LLM-Enhanced LCA: LLM-Assisted Material Interpretation and ELCD Process Matching

This repository contains the implementation and reproducibility materials for an LLM-assisted workflow for upfront whole-building life-cycle assessment (WBLCA), with particular emphasis on material interpretation and environmental process matching.

The current controlled benchmark evaluates whether open-weight large language models can support three tasks that occur before embodied-carbon calculation:

1. material normalization from construction Bill of Materials (BOM) descriptions;
2. selection of a suitable environmental process from a retrieved ELCD candidate set; and
3. classification of the result as Direct, Proxy, or Review Required.

The benchmark does **not** ask the evaluated LLMs to generate emission factors, GWP values, EPDs, or environmental process UUIDs. All selectable environmental processes come from a fixed process catalog exported from openLCA.

---

## 1. Repository Purpose

The broader framework is intended to support automated or semi-automated embodied-carbon screening from construction material inventories.

For the controlled four-model experiment included in this repository, the LLM role is deliberately restricted to:

- interpreting construction material descriptions;
- normalizing material names;
- evaluating retrieved ELCD candidate processes;
- selecting the most defensible candidate when one is available;
- distinguishing Direct matches from Proxy matches; and
- routing unmatched materials to Review Required.

Environmental calculations are downstream of this benchmark and are not part of the model-comparison experiment.

---

## 2. Controlled Four-Model Benchmark

Four open-weight instruction-tuned models are evaluated:

| Model | Hugging Face checkpoint |
|---|---|
| Llama 3.1 8B Instruct | `meta-llama/Llama-3.1-8B-Instruct` |
| Qwen2.5 7B Instruct | `Qwen/Qwen2.5-7B-Instruct` |
| DeepSeek LLM 7B Chat | `deepseek-ai/deepseek-llm-7b-chat` |
| Mistral 7B Instruct v0.3 | `mistralai/Mistral-7B-Instruct-v0.3` |

The same frozen reference set, retrieval procedure, candidate pool, instructions, and inference configuration are used for all four models.

---

## 3. Expert Reference Set

The benchmark uses a reconciled 35-item expert reference set derived from the three building case studies used in the study.

The final reference distribution is:

| Reference class | Number of materials |
|---|---:|
| Direct | 13 |
| Proxy | 15 |
| Review Required | 7 |
| **Total** | **35** |

### Definitions

**Direct**

A suitable ELCD process represents the original material or product sufficiently well for the screening application.

**Proxy**

An exact/direct process is not available, but a technically defensible ELCD substitute is available and selected.

**Review Required**

No supplied ELCD process is considered sufficiently defensible. No final ELCD process UUID is assigned.

Only Review Required cases are eligible for a later fallback or manual-review stage outside the controlled model-matching benchmark.

---

## 4. Independent Expert Review

The expert reference set was developed before model scoring.

Two reviewers independently evaluated the 35 BOM materials using the same exported ELCD catalog.

Before reconciliation, expert agreement was:

| Measure | Agreement |
|---|---:|
| Normalized material | 25/35 = 71.4% |
| Selected ELCD process | 23/35 = 65.7% |
| Match type | 29/35 = 82.9% |
| Full agreement across all three fields | 17/35 = 48.6% |

Disagreements were reconciled to produce one frozen reference answer for each material before the four LLMs were evaluated.

The expert workbook is located at:

```text
ELCD_Check/expert_reference/LLM_LCA_Expert_Reference_Set_With_ELCD.xlsx
```

The frozen machine-readable benchmark reference is generated using:

```bash
python scripts/prepare_benchmark_reference.py
```

and saved as:

```text
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

---

## 5. ELCD Process Catalog

The benchmark uses a fixed catalog exported from the ELCD 3.2 database active in openLCA.

Catalog location:

```text
ELCD_Check/ELCD_Process_Catalog.xlsx
```

The current catalog contains:

```text
608 processes
```

Each process contains at least:

- process UUID;
- process name;
- available location information; and
- process type.

The catalog contains both LCI-result and unit-process descriptors.

The catalog is exported through the openLCA IPC interface using:

```text
scripts/export_openlca_process_catalog.py
```

Before exporting the catalog, the intended ELCD 3.2 database must be active in openLCA and the IPC server must be running.

The stored database label documents the study configuration. It does not independently switch or verify the database active in openLCA.

See:

```text
ELCD_Check/README.md
```

for the detailed catalog and expert-reference workflow.

---

## 6. Deterministic Candidate Retrieval

The four-model benchmark does not ask an LLM to search all 608 ELCD processes directly.

Instead, a deterministic retrieval stage first produces a small candidate set.

The retrieval method is:

```text
Character n-gram TF-IDF
Analyzer: char_wb
N-gram range: 3–5
Query source: original BOM description only
Candidate pool size: 5
```

Human reference labels and expert-normalized material names are **not used** in candidate retrieval.

For the frozen benchmark:

```text
Matched Direct/Proxy materials = 28
Expert process recovered in Top-5 = 21
Top-5 candidate recall = 21/28 = 75.0%
```

The deterministic TF-IDF Top-1 baseline is:

```text
11/28 = 39.3%
```

This Top-1 result provides a non-LLM baseline against which the LLM reranking/selection stage can be compared.

---

## 7. Candidate Presentation to the LLM

TF-IDF is used to determine which five processes belong in the candidate pool.

However, the LLM is **not shown**:

- the TF-IDF similarity score; or
- the original TF-IDF ranking position.

Before presentation to the model, the five candidates are placed in a deterministic shuffled order.

The same candidate set and same presentation order are used for all four models.

This separates:

```text
Retrieval
    ↓
TF-IDF identifies the candidate set

from

Selection
    ↓
LLM independently evaluates the supplied candidate processes
```

This design allows LLM selection performance to be compared with the TF-IDF-only Top-1 baseline without encouraging the LLM simply to accept the retriever's first-ranked process.

---

## 8. LLM Output Task

For each material, the model is instructed to:

1. produce a concise normalized material name;
2. rank up to three supplied process UUIDs;
3. classify the result as:
   - Direct,
   - Proxy, or
   - Review Required.

The model may only use UUIDs contained in the supplied candidate set.

It may not invent:

- process UUIDs;
- emission factors;
- GWP values;
- EPDs;
- database records; or
- environmental values.

For Review Required cases, no selected process UUID is returned.

---

## 9. Final Benchmark Configuration

The formal benchmark configuration is:

| Parameter | Setting |
|---|---|
| Benchmark materials | 35 |
| Candidate pool | 5 |
| LLM ranked output | Top 3 |
| Temperature | 0.0 |
| Decoding | Greedy |
| Sampling | `do_sample=False` |
| Maximum new tokens | 256 |
| Quantization | 4-bit NF4 |
| Main seed | 42 |
| Main runs per material | 1 |
| Repeatability subset | 12 materials |
| Additional repeatability pass | 1 |

The repeatability subset contains:

```text
4 Direct
4 Proxy
4 Review Required
```

### Number of formal responses

Main experiment:

```text
35 materials × 4 models = 140 responses
```

Repeatability experiment:

```text
12 materials × 4 models × 1 additional pass = 48 responses
```

Total:

```text
140 + 48 = 188 formal LLM responses
```

---

## 10. Why Only One Main Run?

The formal benchmark uses deterministic greedy decoding at temperature 0.0.

Under this configuration, five repeated inference passes for every material would provide limited additional information while substantially increasing computation.

Instead, repeatability is evaluated using one additional pass on a fixed balanced 12-item subset.

Repeatability reporting includes response validity so that repeated invalid outputs are not counted as successful agreement.

Because greedy decoding is used, this analysis should be interpreted primarily as test-retest or deterministic inference stability rather than stochastic sampling variability.

---

## 11. Evaluation Metrics

The benchmark separates retrieval performance from LLM performance.

### Material normalization

Reported metrics include:

- exact normalization accuracy;
- normalized-name similarity.

### Candidate retrieval

Reported independently of the LLM:

- candidate-pool recall;
- TF-IDF Top-1 baseline.

### Process ranking

Reported for matched Direct/Proxy rows:

- Top-1 ranking accuracy;
- Top-3 ranking recall;
- mean reciprocal rank.

### Final process selection

Two process-selection metrics are particularly important.

**Overall process-selection accuracy**

Evaluates process selection over all matched Direct/Proxy reference materials.

This metric includes failures caused by both candidate retrieval and LLM selection.

**Conditional process-selection accuracy**

Evaluates the LLM only for cases in which the expert-reference process was actually present in the candidate pool.

This separates LLM selection ability from upstream retrieval failure.

### Match classification

Reported metrics include:

- Direct/Proxy/Review Required accuracy;
- Review Required binary accuracy;
- Review Required precision;
- Review Required recall;
- Review Required F1.

### Reliability

The workbooks also report:

- structured-output validity;
- failed-response rate;
- generated token count;
- token-limit indicators;
- inference time; and
- repeatability/stability.

---

## 12. Repository Structure

The relevant benchmark structure is:

```text
LLM-Screening-LCA/
│
├── ELCD_Check/
│   ├── ELCD_Process_Catalog.xlsx
│   ├── README.md
│   └── expert_reference/
│       └── LLM_LCA_Expert_Reference_Set_With_ELCD.xlsx
│
├── Four_Models/
│   ├── README.md
│   ├── Input/
│   │   └── LLM_Model_Evaluation_Reference_Set.xlsx
│   └── Output/
│       ├── llama/
│       ├── qwen/
│       ├── deepseek/
│       ├── mistral/
│       ├── combined/
│       ├── repeatability/
│       └── smoke/
│
├── scripts/
│   ├── export_openlca_process_catalog.py
│   ├── prepare_benchmark_reference.py
│   ├── benchmark_four_llms.py
│   └── run_four_llm_benchmark.py
│
├── docs/
│   └── REPRODUCIBILITY.md
│
├── requirements-benchmark.txt
└── README.md
```

---

## 13. Installation

A CUDA-capable environment is recommended for the four-model benchmark.

For Google Colab:

```bash
pip install -r requirements-benchmark.txt
```

The benchmark uses Hugging Face Transformers and 4-bit NF4 loading.

Access to gated model repositories must be configured before inference where required.

For example, Llama access may require an approved Hugging Face account and authentication token.

---

## 14. Prepare the Frozen Reference Set

Generate the benchmark input from the reconciled expert workbook:

```bash
python scripts/prepare_benchmark_reference.py
```

This creates:

```text
Four_Models/Input/LLM_Model_Evaluation_Reference_Set.xlsx
```

The preparation script validates the expert reference before freezing it.

---

## 15. Validate Inputs Without Loading an LLM

Run:

```bash
python scripts/benchmark_four_llms.py --check-inputs
```

For the current reference set, the expected summary is approximately:

```text
Catalog processes: 608
Reference rows: 35
Matched Direct/Proxy rows: 28
Review Required rows: 7
Candidate-pool recall at Top-5: 21/28 (75.0%)
TF-IDF Top-1 baseline: 11/28 (39.3%)
```

This step does not load an LLM.

---

## 16. Smoke Test

Before the formal benchmark, run:

```bash
python scripts/run_four_llm_benchmark.py --smoke
```

The smoke test uses a small subset and writes to a separate smoke-output directory so that smoke-test results do not overwrite formal benchmark workbooks.

---

## 17. Run the Complete Four-Model Benchmark

Run:

```bash
python scripts/run_four_llm_benchmark.py
```

The runner:

1. prepares/validates the frozen reference;
2. validates the ELCD catalog;
3. runs Llama;
4. runs Qwen;
5. runs DeepSeek;
6. runs Mistral;
7. runs the fixed repeatability subset; and
8. creates combined Excel results.

Each model is executed in a separate subprocess so GPU memory can be released before the next model is loaded.

---

## 18. Output Workbooks

Formal model results are written under:

```text
Four_Models/Output/
```

Typical structure:

```text
Four_Models/Output/
├── llama/
│   └── benchmark_results.xlsx
├── qwen/
│   └── benchmark_results.xlsx
├── deepseek/
│   └── benchmark_results.xlsx
├── mistral/
│   └── benchmark_results.xlsx
├── combined/
│   └── four_model_comparison.xlsx
└── repeatability/
    ├── llama/
    ├── qwen/
    ├── deepseek/
    ├── mistral/
    └── repeatability_check.xlsx
```

Smoke-test workbooks are stored separately under:

```text
Four_Models/Output/smoke/
```

---

## 19. Reproducibility Metadata

Each formal model workbook records available provenance information such as:

- exact model checkpoint;
- model revision;
- tokenizer revision;
- benchmark script version;
- prompt hash;
- ELCD catalog hash;
- frozen benchmark-reference hash;
- candidate-pool configuration;
- candidate-presentation method;
- seed;
- temperature;
- decoding configuration;
- maximum generated tokens;
- quantization configuration;
- GPU information;
- CUDA version;
- Python version;
- PyTorch version;
- Transformers version;
- Accelerate version;
- BitsAndBytes version;
- inference time;
- generated-token count; and
- token-limit status.

See:

```text
docs/REPRODUCIBILITY.md
```

for additional details.

---

## 20. Important Methodological Interpretation

A failed retrieval and a failed LLM selection are not the same error.

For example:

```text
Original BOM material
        ↓
TF-IDF retrieval
        ↓
Is expert process in Top-5?
        ↓
YES                         NO
 ↓                           ↓
LLM can select it       LLM cannot select the
                        exact expert UUID
```

For this reason, both overall and conditional process-selection metrics are reported.

The benchmark therefore evaluates the complete pipeline while still distinguishing the contribution and limitation of its retrieval component.

---

## 21. Current Retrieval Limitation

Increasing the candidate pool beyond five did not improve expert-reference process coverage for the current dataset.

The Top-5 pool recovered:

```text
21/28 = 75.0%
```

of matched reference processes.

A diagnostic Top-20 retrieval test recovered the same:

```text
21/28 = 75.0%
```

The remaining unmatched reference-process relationships are primarily semantic proxy relationships rather than simple lexical matches.

For this reason, Top-5 is retained as the formal benchmark configuration.

No post-hoc synonym dictionary derived from the expert answers is introduced, because doing so could leak reference information into the retrieval stage.

---

## 22. Scope

The controlled model benchmark evaluates LLM-assisted material interpretation and environmental-process matching.

It should not be interpreted as an evaluation of:

- complete LCA accuracy;
- environmental database quality;
- product-specific EPD accuracy;
- geographic representativeness of ELCD;
- downstream openLCA impact calculations; or
- the accuracy of LLM-generated emission factors.

These are separate methodological questions.

---

## 23. Reproducibility

Detailed instructions are provided in:

```text
docs/REPRODUCIBILITY.md
```

The repository is intended to preserve the distinction between:

```text
Expert reference creation
        ↓
Deterministic retrieval
        ↓
LLM interpretation and selection
        ↓
Downstream environmental calculation
```

This separation is central to the experimental design.
