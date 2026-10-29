# Operational Runbook: Hillclimbing with Autoraters Challenge Lab

This runbook provides the exact step-by-step commands, analytical reasoning, and file modifications executed to complete the **Hillclimbing with Autoraters** Challenge Lab (`qwiklabs-gcp-04-6b58eac25491`) with a **100% score (5.0000 / 5.0000)**.

---

## Step 1: Environment Setup & Dependency Installation

Instead of compiling Python 3.13 from source via `pyenv` (which takes ~8–10 minutes), use `uv` to install a pre-built Python 3.13 environment and all required packages in ~10 seconds:

```bash
export PROJECT_ID=$(gcloud config get-value project)

# 1. Download lab codebase from Cloud Storage
cd ~
gcloud storage cp -r gs://${PROJECT_ID}-bucket/autoraters .

# 2. Create Python 3.13 virtual environment & install dependencies
uv python install 3.13
uv venv ~/.venv313 --python 3.13
uv pip install --python ~/.venv313/bin/python -r ~/autoraters/requirements.txt
export PATH="$HOME/.venv313/bin:$PATH"

# 3. Create .env file for Vertex AI & ADK
cd ~/autoraters
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF
```

---

## Step 2: Inspect the Agent & Generate Baseline Reports (Tasks 2 & 3)

### Inspect Agent Architecture
- [`re_analyst/agent.py`](./re_analyst/agent.py): Defines `root_agent` (`LlmAgent` using `gemini-3.5-flash`), wraps three skills via `SkillToolset` (`analyze-opportunity`, `analyze-flood-risk`, `analyze-tax-implications`), and registers custom Python tools (`get_lot_data`, `get_zoning_code_data`, `calculate_financial_projections`).
- [`re_analyst/tools/tools.py`](./re_analyst/tools/tools.py): Queries [`tools_data/lots_database.json`](./tools_data/lots_database.json) and [`tools_data/zoning_codes_database.json`](./tools_data/zoning_codes_database.json).

### Run Batch Generation (Dry-Run)
```bash
cd ~/autoraters
python3 -m eval_harness.run_agent --dry-run
```
This reads [`eval_harness/prompts.json`](./eval_harness/prompts.json) and populates [`runs/agent_outputs_from_dry_run.json`](./runs/agent_outputs_from_dry_run.json) for 32 vacant lots (`LOT-1020` through `LOT-1051`).

---

## Step 3: Evaluate with LLM-as-a-Judge (Task 4)

### Run Multi-Pass Evaluation (Dry-Run)
```bash
cd ~/autoraters
python3 -m eval_harness.run_geap_eval --dry-run --num-passes 15
```
This simulates $k = 15$ evaluation passes across all 19 rubric criteria defined in [`eval_harness/criterion_rubrics.py`](./eval_harness/criterion_rubrics.py) and saves the output to `runs/geap_calls_and_responses_from_dry_run.json`.

---

## Step 4: Statistical Reliability Analysis & Criterion Selection (Task 5)

### 1. Run Summary Metrics Across All 19 Criteria
```bash
cd ~/autoraters
python3 -m eval_harness.all_criteria_analysis --input-file runs/geap_calls_and_responses_for_analysis.json
```

### 2. Inspect Candidate Criteria in Detail
```bash
python3 -m eval_harness.single_criterion_analysis --input-file runs/geap_calls_and_responses_for_analysis.json --criterion monthly-tax-holding
python3 -m eval_harness.single_criterion_analysis --input-file runs/geap_calls_and_responses_for_analysis.json --criterion flood-zone-flagging
python3 -m eval_harness.single_criterion_analysis --input-file runs/geap_calls_and_responses_for_analysis.json --criterion neighboring-flood-risk
```

### 3. Why Other High-Score Criteria Are Eliminated
| Criterion | Reason for Rejection |
| :--- | :--- |
| `general-tone`, `logical-flow`, `professionalism`, `readability` | **Homogeneity Trap (`sd_report_means = 0.0000`)**: Every report scored `5` on every pass. Zero discriminative signal. |
| `data-density`, `professional-tone` | **Near-Zero Variance ($\alpha \approx 0.498$)**: Over 99% of ratings are `5`, with only 1–2 random drops to `4`. |
| `environmental-esa` | **High Single-Report Instability (`max_range_report = 3`)**: Despite $\alpha = 0.897$, pairwise agreement is only `79.10%` and `LOT-1040` fluctuates between `2` and `5`. |
| `cost-per-acre`, `far-utilization`, `formatting-headers`, `holding-period`, `market-comparables`, `price-appreciation`, `setback-impact`, `soil-suitability`, `utility-availability`, `verdict-clarity`, `zoning-compliance` | **Low Intra-Rater Reliability ($\alpha < 0.72$, `max_range_report` of `3` or `4`)**: Severe rating noise across passes. |

### 4. Selected Reliable Criteria & Submission
The 3 reliable criteria with high Krippendorff's $\alpha$, high pairwise agreement, low max single-report range ($\le 1$), and non-zero between-report variance are:
- `monthly-tax-holding` ($\alpha = 0.9968$, `%_agree = 99.17%`, `max_range_report = 1`)
- `flood-zone-flagging` ($\alpha = 0.8971$, `%_agree = 94.41%`, `max_range_report = 1`)
- `neighboring-flood-risk` ($\alpha = 1.0000$, `%_agree = 100.00%`, `max_range_report = 0`, `sd_report_means = 0.1768`)

```bash
cd ~/autoraters
echo -n "monthly-tax-holding,flood-zone-flagging,neighboring-flood-risk" > best_criteria_submission.txt
gcloud storage cp best_criteria_submission.txt gs://${PROJECT_ID}-bucket/best_criteria_submission.txt
```

---

## Step 5: Hill-Climbing Agent Skills (Task 6)

### Target Criteria & Passing Threshold
- **Target Criteria:** `cost-per-acre` and `neighboring-flood-risk`
- **Target Score:** Mean rating **> 4.0** on both criteria via `python3 -m eval_harness.run_hillclimb`

### Root Cause Diagnosis
1. **`cost-per-acre` (Baseline Mean: `2.7042`)**
   - Rubric requires: (a) accurate calculation (`Last Sale Price / Acreage`), (b) explicit inclusion in the report, and (c) **contextualization as a key valuation metric** (benchmarked against adjacent parcels' cost per acre and assessed land value per acre).
   - Baseline `analyze-opportunity/SKILL.md` only mentioned calculating cost per acre in passing without instructing the agent to show the formula, highlight it as a core metric, or compare it against neighboring lots.
2. **`neighboring-flood-risk` (Baseline Mean: `1.0312`)**
   - Rubric requires: **Detailed comparative flood risk analysis** between the target lot and its neighboring parcels.
   - Baseline `analyze-flood-risk/SKILL.md` did not grant access to `get_lot_data` in `allowed-tools` and never instructed the agent to look up the `adjacent_parcels` field in `lots_database.json` or call `get_lot_data` for each adjacent lot ID.

### Skill File Updates

#### 1. [`re_analyst/skills/analyze-opportunity/SKILL.md`](./re_analyst/skills/analyze-opportunity/SKILL.md)
Updated to require:
- Calling `get_lot_data` for the target lot **and** every lot ID in `adjacent_parcels`.
- Dedicated section **"Cost Per Acre Valuation & Comparative Benchmark Analysis"** showing the exact formula (`Last Sale Price / Acreage`), exact dollar figure, comparison against assessed land value per acre, and a comparative table against all adjacent parcels' cost per acre and flood zones.
- Dedicated section **"Neighboring Parcel Comparative Flood Risk Analysis"** synthesizing hydrological and flood zone comparisons across all adjacent lots.

#### 2. [`re_analyst/skills/analyze-flood-risk/SKILL.md`](./re_analyst/skills/analyze-flood-risk/SKILL.md)
Updated to:
- Add `get_lot_data` to `allowed-tools: get_lot_data calculate_financial_projections`.
- Require calling `get_lot_data` for every parcel ID listed in `adjacent_parcels`.
- Produce a dedicated **"Neighboring Parcel Comparative Flood Risk Analysis"** section with a side-by-side markdown table (Lot ID, Relationship, Acreage, Last Sale Price, Cost/Acre, FEMA Flood Zone, Topography, Drainage/Runoff Impact) and detailed narrative comparing cross-parcel hydrological risk, insurance/lending implications, and mitigation engineering.

#### 3. [`re_analyst/skills/analyze-tax-implications/SKILL.md`](./re_analyst/skills/analyze-tax-implications/SKILL.md)
Updated to:
- Add `get_lot_data` to `allowed-tools` and explicitly calculate annual and monthly tax holding costs (`Annual Tax / 12`) plus tax basis per acre alongside cost-per-acre metrics.

### Execute Hill-Climb Evaluation
```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```

### Final Verification Output
- **Output File:** [`runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json`](./runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json)
- **Vertex AI Experiment:** `hillclimb-20261029-082752`
- **Final Scores Across All 10 Evaluated Lots × 3 Passes (30 Evaluations per Criterion):**
  - **`cost-per-acre`:** `5.0000` (30/30 ratings = `5`)
  - **`neighboring-flood-risk`:** `5.0000` (30/30 ratings = `5`)
