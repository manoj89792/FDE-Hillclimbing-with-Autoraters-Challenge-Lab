# FDE Hillclimbing with Autoraters — Challenge Lab Runbook & Solution Guide

Complete step-by-step runbook, statistical analysis, root-cause diagnosis, and solution implementation for the **Hillclimbing with Autoraters Challenge Lab** on Google Cloud (Agent Development Kit + Vertex AI Gen AI Evaluation Service / Gemini Enterprise Agent Platform).

---

## 📌 Table of Contents
1. [Lab Overview & Architecture](#1-lab-overview--architecture)
2. [Repository Structure](#2-repository-structure)
3. [Task 1: Environment Setup (Fast Track & Standard Track)](#3-task-1-environment-setup-fast-track--standard-track)
4. [Task 2: Run, Test, and Understand the Agent](#4-task-2-run-test-and-understand-the-agent)
5. [Task 3: Generate Agent Evaluation Traces](#5-task-3-generate-agent-evaluation-traces)
6. [Task 4: Evaluate Agent Responses with LLM-as-a-Judge (GEAP)](#6-task-4-evaluate-agent-responses-with-llm-as-a-judge-geap)
7. [Task 5: Analyze Autorater Intra-Rater Reliability (Check My Progress #1)](#7-task-5-analyze-autorater-intra-rater-reliability-check-my-progress-1)
   - [All 19 Criteria Metrics Table](#all-19-criteria-metrics-table)
   - [How to Distinguish Useful Metrics vs. Distractors](#how-to-distinguish-useful-metrics-vs-distractors)
   - [Avoiding the Homogeneity Trap](#avoiding-the-homogeneity-trap)
   - [Selecting the 3 Best Criteria & Uploading to GCS](#selecting-the-3-best-criteria--uploading-to-gcs)
8. [Task 6: Hill Climb Agent Skills to Raise Autorater Scores (Check My Progress #2)](#8-task-6-hill-climb-agent-skills-to-raise-autorater-scores-check-my-progress-2)
   - [Root Cause Diagnosis of Baseline Deficiencies](#root-cause-diagnosis-of-baseline-deficiencies)
   - [Skill Modifications (`SKILL.md` Changes)](#skill-modifications-skillmd-changes)
   - [Running `run_hillclimb.py` & Verifying 5.0/5.0 Results](#running-run_hillclimbpy--verifying-5050-results)
9. [Quick Copy-Paste Execution Script (Full Lab Automation)](#9-quick-copy-paste-execution-script-full-lab-automation)

---

## 1. Lab Overview & Architecture

In this challenge lab, you prototype and optimize `re_analyst`, an **Agent Development Kit (ADK)** agent powered by **Gemini (`gemini-3.5-flash`)** that generates real estate investment opportunity memorandums for vacant lots.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        1. Agent Trace Generation                           │
│  eval_harness/hillclimb_lots.txt ──► re_analyst (ADK Agent + Skills/Tools) │
│                                           │                                │
│                                           ▼                                │
│                            re_analyst/hillclimb_X.evalset.json             │
└───────────────────────────────────────────┬────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                   2. LLM-as-a-Judge Evaluation (GEAP)                      │
│  eval_harness/rubrics.json + EvalSet ──► Vertex AI EvalTask (Pointwise)    │
│                                           │                                │
│                                           ▼                                │
│              runs/hillclimb_X/geap_calls_and_responses_hillclimb_X.json    │
└───────────────────────────────────────────┬────────────────────────────────┘
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│             3. Reliability Analysis & Iterative Hill Climbing              │
│  • Krippendorff's Alpha & Pairwise Agreement across k=15 passes            │
│  • Diagnose judge rationales (explain_criteria_results.py)                 │
│  • Refine ADK skills (SKILL.md) until target criteria mean score > 4.0     │
└────────────────────────────────────────────────────────────────────────────┘
```

### Key Objectives
1. **Interpret inter-/intra-rater reliability metrics** (`Krippendorff's Alpha`, `Mean Pairwise Percent Agreement`, `Widest Range Single Report`, `StdDev of Report Means`).
2. **Distinguish useful metrics from distractors** and avoid the **homogeneity trap** when assessing autorater self-consistency.
3. **Diagnose agent deficiencies** using LLM judge explanations, rubric definitions, and ADK tool data.
4. **Hill-climb agent skills** (`analyze-opportunity`, `analyze-flood-risk`, `analyze-tax-implications`) to achieve a mean score **> 4.0** (we achieve **5.0000**) on both `cost-per-acre` and `neighboring-flood-risk`.

---

## 2. Repository Structure

```text
.
├── README.md                                  # Complete Step-by-Step Guide & Runbook
├── RUNBOOK.md                                 # Quick-Reference Operational Runbook
├── best_criteria_submission.txt               # Task 5 submission file uploaded to GCS
├── requirements.txt                           # Python 3.13 dependencies (google-adk, vertexai, krippendorff)
├── eval_harness/
│   ├── all_criteria_analysis.py               # Computes summary & reliability stats across all 19 criteria
│   ├── single_criterion_analysis.py           # Displays N x k grades matrix & margins for 1 criterion
│   ├── explain_criteria_results.py            # Prints LLM judge rationales filtered by max score
│   ├── create_re_agent_evalset.py             # Runs ADK agent concurrently over target lots
│   ├── run_geap_eval.py                       # Bridges ADK EvalSet traces to Vertex AI EvalTask
│   ├── run_hillclimb.py                       # End-to-end hill-climbing runner (k=1 on target criteria)
│   ├── metrics_calc.py                        # Krippendorff's Alpha (interval) & pairwise agreement math
│   ├── hillclimb_lots.txt                     # 10 target Lot IDs for hill climbing
│   ├── rubrics.json                           # 19 evaluation criteria and 1-5 rating scales
│   └── utils.py                               # Helper utilities for loading traces and formatting tables
├── re_analyst/
│   ├── agent.py                               # Root ADK Agent (`re_opportunity_analyst`) & SkillToolset
│   ├── skills/
│   │   ├── analyze-opportunity/SKILL.md       # [UPDATED] Core investigation & cost-per-acre benchmarking skill
│   │   ├── analyze-flood-risk/SKILL.md        # [UPDATED] Subject & neighboring parcel flood/hydrology skill
│   │   └── analyze-tax-implications/SKILL.md  # [UPDATED] Official APN tax & monthly holding cost skill
│   └── tools/
│       ├── lot_tool.py                        # `get_lot_data(plot_id)` reading lots_database.json
│       └── tax_tool.py                        # `get_tax_assessment_data(parcel_apn)` reading tax_database.json
├── tools_data/
│   ├── lots_database.json                     # Property records (acreage, zoning, flood_zone, adjacent_parcels)
│   └── tax_database.json                      # Official county tax records by APN
└── runs/
    └── hillclimb_1/
        └── geap_calls_and_responses_hillclimb_1.json  # Verified 5.0000 / 5.0000 evaluation output
```

---

## 3. Task 1: Environment Setup (Fast Track & Standard Track)

### Step 1.1: Download the Lab Code from Cloud Storage
Replace `<PROJECT_ID>` with your Qwiklabs GCP Project ID (e.g., `qwiklabs-gcp-04-6b58eac25491`):

```bash
export PROJECT_ID=$(gcloud config get-value project)
cd ~
gcloud storage cp -r gs://${PROJECT_ID}-bucket/autoraters .
```

### Step 1.2: Install Python 3.13 & Dependencies
> **⚡ Pro Tip (Fast Track with `uv` — takes ~7 seconds instead of 10 minutes):**
> Because `krippendorff-aleph-alpha==0.1.17` requires `Python >= 3.13`, compiling Python 3.13 from source via `pyenv` takes 5–10 minutes. Cloud Shell has `uv` pre-installed, which downloads a prebuilt Python 3.13 binary and installs all 104 packages in **under 8 seconds**:

```bash
uv python install 3.13
uv venv ~/.venv313 --python 3.13
uv pip install --python ~/.venv313/bin/python -r ~/autoraters/requirements.txt
export PATH="$HOME/.venv313/bin:$PATH"
echo 'export PATH="$HOME/.venv313/bin:$PATH"' >> ~/.bashrc
```

*(Optional: If you prefer the standard `pyenv` track from the lab instructions, run `curl https://pyenv.run | bash`, `pyenv install 3.13`, `pyenv global 3.13`, and `python3 -m pip install -r autoraters/requirements.txt`.)*

### Step 1.3: Configure `.env` and Project Settings
```bash
cd ~/autoraters
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF

gcloud config set project ${PROJECT_ID}
```

---

## 4. Task 2: Run, Test, and Understand the Agent

You can interactively inspect the ADK agent using the ADK Web UI:

```bash
cd ~/autoraters
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```

- Open `http://127.0.0.1:8000`, select **`re_analyst`**, and test with:
  ```text
  Please evaluate investment suitability for lot LOT-8472 and generate a report.
  ```

### How `re_analyst` Works
- **`re_analyst/agent.py`**: Initializes `re_opportunity_analyst` with a `SkillToolset` containing three skills (`analyze-opportunity`, `analyze-flood-risk`, `analyze-tax-implications`) and two function tools (`get_lot_data`, `get_tax_assessment_data`).
- **`tools_data/lots_database.json`**: Contains parcel attributes including `plot_id`, `parcel_apn`, `acreage`, `zoning`, `flood_zone`, `last_sale_price`, `estimated_annual_taxes`, `environmental_status`, `utility_access`, `topography`, and **`adjacent_parcels`** (a list of neighboring Lot IDs, e.g. `["LOT-8471", "LOT-8473"]`).
- **`tools_data/tax_database.json`**: Keyed by `parcel_apn`, containing `annual_tax_amount`, `assessed_land_value`, and `total_assessed_value`.

---

## 5. Task 3: Generate Agent Evaluation Traces

To generate the evaluation dataset across the 10 properties listed in `eval_harness/hillclimb_lots.txt`:

```bash
cd ~/autoraters
python3 -m eval_harness.create_re_agent_evalset \
  --lots-file eval_harness/hillclimb_lots.txt \
  --out ./re_analyst/hillclimb_lots.evalset.json \
  --agent-dir ./re_analyst
```

---

## 6. Task 4: Evaluate Agent Responses with LLM-as-a-Judge (GEAP)

### Step 6.1: Dry Run (Local Prompt Assembly)
```bash
cd ~/autoraters
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --dry-run \
  --k 3
```

### Step 6.2: Live Cloud Evaluation (Optional before Task 6)
```bash
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --k 3
```
> **⚠️ Important Note for Grading:** In Task 6, the lab grader checks the **most recent evaluation run** in Vertex AI Experiments. Always ensure your final Vertex AI evaluation run in the lab is a successful `run_hillclimb` execution where both target criteria score above `4.0`.

---

## 7. Task 5: Analyze Autorater Intra-Rater Reliability (Check My Progress #1)

In Task 5, we analyze `runs/geap_calls_and_responses_for_analysis.json`, which contains **32 lot IDs ($N=32$) graded across 19 criteria over 15 repeated evaluation passes ($k=15$)**.

### Step 7.1: Run Single-Criterion and All-Criteria Analysis
```bash
cd ~/autoraters

# Deep-dive into a single criterion:
python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion environmental-esa

# Summary across all 19 criteria:
python3 -m eval_harness.all_criteria_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json
```

### All 19 Criteria Metrics Table

| `criterion_id` | `criterion_name` | `mean` | `min` | `max` | `max range report` | `max range pass` | `sd report means` | `sd pass means` | `pairwise %_agree` | `alpha` | Verdict / Analysis |
| :--- | :--- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | :--- |
| **`monthly-tax-holding`** | Monthly Tax Holding Cost Calculation | 2.2764 | 2 | 5 | **1** | 3 | **0.8872** | 0.0228 | **99.17%** | **0.997** | ✅ **Selected (#1)**: Highest true reliability ($\alpha=0.997$), high report variance (`sd=0.8872`), max report range = 1 |
| **`flood-zone-flagging`** | Precise Flood Zone Flagging | 4.8447 | 3 | 5 | **1** | 2 | **0.3357** | 0.0256 | **94.41%** | **0.897** | ✅ **Selected (#2)**: High reliability ($\alpha=0.897$, 94.41% agreement), good variance (`sd=0.3357`), max report range = 1 |
| **`neighboring-flood-risk`** | Neighboring Property Flood Risk Analysis | 1.0316 | 1 | 2 | **0** | 1 | **0.1768** | 0.0009 | **100.00%** | **1.000** | ✅ **Selected (#3)**: 100% consistent across all 15 passes (`max_range_report=0`), distinguishes `LOT-1022` (score 2) from others (score 1) |
| `environmental-esa` | Environmental Status (Phase I ESA) Evaluation | 4.2772 | 2 | 5 | **3** | 3 | 0.7042 | 0.0991 | 79.10% | 0.897 | ⚠️ High alpha, but severe single-report disagreement (`max_range_report = 3` on `LOT-1040`, pairwise agreement only 79.10%) |
| `risk-assessment` | Comprehensive Risk Assessment | 4.9532 | 3 | 5 | 2 | 2 | 0.1843 | 0.0306 | 96.25% | 0.803 | 🟡 Borderline acceptable ($\alpha=0.803$), but `max_range_report = 2` on `LOT-8473` |
| `utility-topography` | Utility & Topography Evaluation | 4.7756 | 4 | 5 | 1 | 1 | 0.3081 | 0.0695 | 82.57% | 0.751 | 🟡 Moderate-to-good ($\alpha=0.751$), slightly below conservative $\alpha \ge 0.80$ cutoff |
| `general-tone` | General Tone Suitability | 5.0000 | 5 | 5 | 0 | 0 | **0** | 0 | 100.00% | 1.000 | ❌ **Homogeneity Trap**: Every report scored 5 on every pass (`sd_report_means = 0`) |
| `logical-flow` | Logical Narrative Flow | 5.0000 | 5 | 5 | 0 | 0 | **0** | 0 | 100.00% | 1.000 | ❌ **Homogeneity Trap**: Zero variance across reports (`sd_report_means = 0`) |
| `professionalism` | Analyst Professionalism | 5.0000 | 5 | 5 | 0 | 0 | **0** | 0 | 100.00% | 1.000 | ❌ **Homogeneity Trap**: Zero variance across reports (`sd_report_means = 0`) |
| `readability` | General Reading Ease | 5.0000 | 5 | 5 | 0 | 0 | **0** | 0 | 100.00% | 1.000 | ❌ **Homogeneity Trap**: Zero variance across reports (`sd_report_means = 0`) |
| `data-density` | Information and Data Density | 4.9979 | 4 | 5 | 1 | 1 | 0.0118 | 0.0081 | 99.58% | 0.499 | ❌ **Homogeneity Trap**: 99.58% pairwise agreement, but $\alpha = 0.499$ due to near-zero variance |
| `professional-tone` | Professional Tone | 4.9958 | 4 | 5 | 1 | 1 | 0.0164 | 0.0110 | 99.17% | 0.498 | ❌ **Homogeneity Trap**: 99.17% pairwise agreement, but $\alpha = 0.498$ due to near-zero variance |
| `persuasiveness` | Document Persuasiveness | 4.8840 | 4 | 5 | 1 | 1 | 0.2022 | 0.0395 | 86.51% | 0.672 | ❌ Low Alpha ($\alpha = 0.672 < 0.80$), subjective rubric |
| `analyst-insight` | Deep Analyst Insights | 4.7294 | 4 | 5 | 1 | 1 | 0.2570 | 0.0576 | 71.59% | 0.639 | ❌ Low Alpha ($\alpha = 0.639 < 0.80$), subjective rubric |
| `vocabulary-choice` | Advanced Real Estate Vocabulary | 4.9349 | 4 | 5 | 1 | 1 | 0.1365 | 0.0391 | 90.83% | 0.623 | ❌ Low Alpha ($\alpha = 0.623 < 0.80$), subjective rubric |
| `core-specifications` | Explicit Plot Core Specifications | 4.3623 | 4 | 5 | 1 | 1 | 0.2648 | 0.0765 | 65.05% | 0.621 | ❌ Low Alpha ($\alpha = 0.621$, 65.05% pairwise agreement) |
| `cost-per-acre` | Acquisition Cost-Per-Acre Calculation | 1.3771 | 1 | 3 | 2 | 2 | 0.2163 | 0.0511 | 58.97% | 0.565 | ❌ Moderate/questionable agreement ($\alpha = 0.565$, 58.97% agreement, range = 2) |
| `report-length` | Report Detail and Length | 4.7622 | 4 | 5 | 1 | 1 | 0.1715 | 0.0721 | 67.20% | 0.549 | ❌ Low Alpha ($\alpha = 0.549$), vague rubric |
| `document-formatting` | Document Readability & Formatting | 4.8361 | 4 | 5 | 1 | 1 | 0.1488 | 0.0487 | 75.22% | 0.548 | ❌ Low Alpha ($\alpha = 0.548$) |

### How to Distinguish Useful Metrics vs. Distractors
- **Useful Metrics for Autorater Reliability:**
  - **Krippendorff's Alpha (`alpha`)**: Global chance-corrected reliability coefficient across all reports and passes. Standard benchmark is $\alpha \ge 0.80$ (and because GEAP uses a 4-vote internal majority vote, we should be conservative and require $\alpha > 0.80$).
  - **Mean Pairwise Percent Agreement (`pairwise %_agree`)**: Measures how often pairs of passes give the exact same score on the same report. Must be interpreted alongside `alpha` and `sd report means`.
  - **Widest Range Single Report (`max range report`)**: Highlights the worst-case instability on a single report across passes (`max - min`). A range of `0` or `1` is tight; a range of `2` or `3` (like `environmental-esa` on `LOT-1040`) indicates severe judge disagreement on specific inputs.
  - **StdDev of Report Means (`sd report means`)**: Crucial check for whether the dataset exhibits score variation across reports. If `sd report means == 0`, the dataset is completely homogeneous and agreement metrics are uninformative.
- **Distractor Metrics (for intra-rater reliability):**
  - `mean`, `min rating`, `max rating`, `max range pass`: Measure agent score levels/spread across reports rather than rater self-consistency on the same report.

### Avoiding the Homogeneity Trap
- Criteria `general-tone`, `logical-flow`, `professionalism`, and `readability` all display `pairwise %_agree = 100.00%` and `alpha = 1`, **but** their `min rating = 5`, `max rating = 5`, and `sd report means = 0`. Because every report received a 5 on every pass, there is zero variance in perceived report quality—falling into the **homogeneity trap**.
- Similarly, `data-density` (`mean = 4.9979`) and `professional-tone` (`mean = 4.9958`) have >99% pairwise agreement, yet a single `4` causes `alpha` to collapse to `~0.498`.

### Selecting the 3 Best Criteria & Uploading to GCS
Write `monthly-tax-holding,flood-zone-flagging,neighboring-flood-risk` to `best_criteria_submission.txt` and upload to your GCS bucket:

```bash
cd ~/autoraters
echo "monthly-tax-holding,flood-zone-flagging,neighboring-flood-risk" > best_criteria_submission.txt
gcloud storage cp best_criteria_submission.txt gs://${PROJECT_ID}-bucket/best_criteria_submission.txt
```
✅ **Click "Check my progress" for Task 5 (`Choose reliable criteria`).**

---

## 8. Task 6: Hill Climb Agent Skills to Raise Autorater Scores (Check My Progress #2)

In Task 6, `run_hillclimb.py` runs `re_analyst` over the 10 lots in `eval_harness/hillclimb_lots.txt` and evaluates two target criteria with $k=1$:
1. **`cost-per-acre`** (*Acquisition Cost-Per-Acre Calculation*)
2. **`neighboring-flood-risk`** (*Neighboring Property Flood Risk Analysis*)

The goal is to achieve a **mean score > 4.0** on both criteria in a single run.

### Root Cause Diagnosis of Baseline Deficiencies

Using `explain_criteria_results.py`, `eval_harness/rubrics.json`, and `tools_data/lots_database.json`, we identify why the baseline agent scored `~1.38` on `cost-per-acre` and `~1.03` on `neighboring-flood-risk`:

1. **Deficiency 1 — `cost-per-acre` (Baseline Mean: ~1.38)**:
   - **Judge Rationale**: The baseline agent reported `last_sale_price` and `acreage` separately, leaving the cost-per-acre implicit. Even if computed, the rubric requires:
     - **Score 4**: *"Correctly computes cost-per-acre with clear derivation, but does not benchmark or emphasize it as a core valuation metric."*
     - **Score 5**: *"Accurately calculates and highlights exact cost-per-acre as a core valuation metric."*
   - **Fix**: Instruct the agent in `analyze-opportunity/SKILL.md` and `analyze-tax-implications/SKILL.md` to explicitly write the formula `Acquisition Cost-Per-Acre = Last Sale Price / Acreage`, compute the exact dollar-per-acre figure, highlight it in bold and in a summary table as a **Core Valuation Metric**, and benchmark it against neighboring parcels' cost-per-acre and county assessed land value per acre.

2. **Deficiency 2 — `neighboring-flood-risk` (Baseline Mean: ~1.03)**:
   - **Judge Rationale**: The baseline `analyze-flood-risk/SKILL.md` only instructed the agent to check the `flood_zone` field of the subject lot. However, every lot in `tools_data/lots_database.json` contains an `"adjacent_parcels"` array (e.g. `["LOT-8471", "LOT-8473"]`), and each adjacent lot has its own record in `lots_database.json` with `flood_zone`, `topography`, and `easement_type`!
   - **Rubric Requirement for Score 5**: *"Exhaustively investigates neighboring property flood zones and critically analyzes potential drainage, runoff, and shared hydrological impacts on the subject parcel's development feasibility."*
   - **Fix**:
     - Grant `get_lot_data` in `analyze-flood-risk/SKILL.md` frontmatter (`adk_additional_tools`).
     - Instruct the agent in both `analyze-opportunity/SKILL.md` and `analyze-flood-risk/SKILL.md` to call `get_lot_data` for **every lot ID listed in `adjacent_parcels`**.
     - Require a dedicated comparative table and parcel-by-parcel hydrological analysis covering each neighbor's exact FEMA flood zone (`Zone AE`, `Zone VE`, `Zone X`), slope grade (`topography`), easements, cross-boundary floodplain/storm surge encroachment, uphill/downhill sheet flow runoff, and impact on site preparation CapEx and development feasibility.

### Skill Modifications (`SKILL.md` Changes)

The updated skill files are located in:
- [`re_analyst/skills/analyze-opportunity/SKILL.md`](re_analyst/skills/analyze-opportunity/SKILL.md)
- [`re_analyst/skills/analyze-flood-risk/SKILL.md`](re_analyst/skills/analyze-flood-risk/SKILL.md)
- [`re_analyst/skills/analyze-tax-implications/SKILL.md`](re_analyst/skills/analyze-tax-implications/SKILL.md)

### Running `run_hillclimb.py` & Verifying 5.0/5.0 Results

Run the hill-climbing harness:

```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```

#### Actual Execution Output:
```text
📂 Created versioned run directory: /home/student_01_6eb461a10530/autoraters/runs/hillclimb_1

🤖 Running agent 're_analyst' over target lots to generate traces...

🚀 Executing 're_analyst' over 10 properties to populate evaluation traces...
-----------------------------------------------------------------
⚡ Starting concurrent execution across 10 properties (max concurrency: 10)...
✅ [5/10] Completed trace for LOT-1001
✅ [6/10] Completed trace for LOT-1002
✅ [1/10] Completed trace for LOT-8472
✅ [10/10] Completed trace for LOT-1006
✅ [3/10] Completed trace for LOT-8473
✅ [9/10] Completed trace for LOT-1005
✅ [8/10] Completed trace for LOT-1004
✅ [4/10] Completed trace for LOT-1000
✅ [2/10] Completed trace for LOT-8471
✅ [7/10] Completed trace for LOT-1003

Cleaning up resources...

☁️ Dispatching single-pass (k=1) evaluation for criteria: ['cost-per-acre', 'neighboring-flood-risk']...
🧪 Vertex AI Experiment Name: hillclimb-20261029-082752
  ☁️ Dispatching EvalTask pass 1/1 to Vertex AI...
All 20 metric requests are successfully computed.
Evaluation Took:15.20671725499983 seconds
✅ Evaluation results saved to -> /home/student_01_6eb461a10530/autoraters/runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json

=========================================
📊 Hill Climbing Results: hillclimb_1
=========================================
 • cost-per-acre             Mean Score: 5.0000
 • neighboring-flood-risk    Mean Score: 5.0000
=========================================
```

✅ **Click "Check my progress" for Task 6 (`Perform hill-climbing automation`).**

---

## 9. Quick Copy-Paste Execution Script (Full Lab Automation)

If you want to run the entire challenge lab from start to finish in a fresh Cloud Shell session, run the following script block:

```bash
# 1. Set Project ID & download lab files
export PROJECT_ID=$(gcloud config get-value project)
cd ~
gcloud storage cp -r gs://${PROJECT_ID}-bucket/autoraters .

# 2. Fast-install Python 3.13 & requirements using uv (~7 seconds)
uv python install 3.13
uv venv ~/.venv313 --python 3.13
uv pip install --python ~/.venv313/bin/python -r ~/autoraters/requirements.txt
export PATH="$HOME/.venv313/bin:$PATH"
echo 'export PATH="$HOME/.venv313/bin:$PATH"' >> ~/.bashrc

# 3. Configure .env
cd ~/autoraters
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF
gcloud config set project ${PROJECT_ID}

# 4. Task 5: Submit the 3 most reliable autorater criteria to GCS
echo "monthly-tax-holding,flood-zone-flagging,neighboring-flood-risk" > ~/autoraters/best_criteria_submission.txt
gcloud storage cp ~/autoraters/best_criteria_submission.txt gs://${PROJECT_ID}-bucket/best_criteria_submission.txt

# 5. Task 6: Update the 3 agent skills in re_analyst/skills/ and run hill climbing
# (Copy the updated SKILL.md files from this repository into ~/autoraters/re_analyst/skills/)
python3 -m eval_harness.run_hillclimb
```
