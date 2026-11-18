# EventX: A Multimodal Benchmark Linking Social Media Posts to Prediction Market Dynamics

**Benchmark:** EventX full corpus (33.9M posts; 155,792 markets). **Evaluation release:** versioned, task-specific T1–T6 subsets. Hosted access details are withheld for anonymous review.

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc/4.0/)

**EventX** is a multimodal benchmark connecting Twitter/X posts to Polymarket prediction-market dynamics. The current release defines six canonical tasks across two tiers: **resolution** (human-reviewed ground truth) and **forecast** (causally constructed labels from later market data). The versioned July 2026 EventX evaluation releases use leakage-audited temporal splits; release identifiers keep their original `kdd.v2` names for reproducibility.

## Quick Start

```bash
pip install -r requirements.txt

# After unpacking the dated July 2026 release bundle
python -m baselines.t1.basic_baseline \
  --local-dir KDD/data/t1_kdd_v2 \
  --output results/predictions.jsonl

# The runner writes majority and random-prior files with explicit prefixes.
# Evaluate the majority baseline against the same frozen release:
python evaluation/evaluate.py --task t1 \
  --predictions results/t1_majority_predictions.jsonl \
  --gold KDD/data/t1_kdd_v2/test.jsonl
```

## Benchmark Overview

| Task | Name | Tier | Output | Primary Metrics |
|------|------|------|--------|-----------------|
| T1 | Market Volume Prediction | Forecast | 3-class interest label | Macro-F1, `high_interest` P@K |
| T2 | Post-to-Market Linking | Resolution | Market ID or `NONE` | Accuracy@1, MRR, `NONE` F1 |
| T3 | Evidence Grading | Resolution | Ordinal 0--5 | Quadratic-weighted kappa, unweighted Cohen's kappa, macro-F1 |
| T4 | Daily Market Movement | Forecast | Direction, magnitude, 1/3/7-day deltas | Dir-Acc, Mag-F1, Spearman rho by horizon |
| T5 | Forward Drift & Persistence | Forecast | 1/3/7-day drift/volume plus decay class | Spearman rho by horizon, decay Macro-F1 |
| T6 | Cross-Market Co-Movement | Forecast | 3-class by 1/3/7-day horizon | Macro-F1, accuracy |

## Tasks

### T1: Conditional Market Volume Prediction
Predict a market's lifetime-volume interest class from market metadata and social signals available before market creation.

- **Release**: `t1.kdd.v2`; 709 train and 275 test markets
- **Labels**: `high_interest`, `moderate_interest`, `low_interest`
- **Input**: Market text/metadata plus the manifest-approved pre-market social feature rung
- **Metrics**: Macro-F1, accuracy, `high_interest` precision@K

### T2: Post-to-Market Linking
Given a post and a frozen contextual candidate set, rank the matching market or select `NONE`.

- **Release**: `t2.gold.r3.contextual.v1`; 544 train, 2,500 validation, and 2,500 test rows
- **Input**: Post text plus candidate questions, resolution rules, and domains
- **Metrics**: Accuracy@1, MRR, `NONE` F1, and candidate recall
- **Protocol**: Thresholds are selected on training/validation only; test labels are sealed

### T3: Evidence Grading and Resolution Potential
Assign an ordinal evidence grade (0--5) to each post-market pair.

- **Grade scale**: 0 (`noise`), 1 (`commentary`), 2 (`speculation`), 3 (`indirect_report`), 4 (`strong_direct`), 5 (`resolving`) - names match the frozen rubric (`t3_annotation_rubric.md`) used by both LLM and human graders
- **Metrics**: Quadratic-weighted and unweighted kappa, macro-F1
- **Ground truth**: silver `final_grade` (`train` split, 279,924 rows, deterministic-checks-then-LLM) vs. a separate 2,687-row human-adjudicated `gold_grade` audit pool (`gold` split). Silver only agrees with gold at kappa_w=0.582, below the project's 0.6 reliability bar - see `T3_Reproducible_Package/metrics.md` Phase 6

### T4: Market Movement Prediction
Predict forward YES-price movement for a market-day post bundle.

- **Release**: `t4.kdd.v2`; 2,875 train, 2,268 validation, and 5,791 test rows
- **Unit**: `(condition_id, bundle_day)` with a decision time at the end of the UTC day
- **Targets**: Direction, magnitude, and continuous deltas at 1, 3, and 7 days
- **Metrics**: Direction accuracy, magnitude macro-F1, and Spearman rho by horizon

### T5: Forward Drift Magnitude and Persistence
Predict the forward behavior of the non-flat T4 market-day subset.

- **Release**: `t5.kdd.v2`; 889 train, 692 validation, and 1,761 test rows
- **Targets**: Drift magnitude and volume multiplier at 1, 3, and 7 days, plus `transient`/`sustained`/`reversal`
- **Metrics**: Spearman rho for every continuous horizon and decay macro-F1
- **Compatibility**: The legacy `t7` config remains an alias for older decay-only data; canonical results use T5

### T6: Cross-Market Co-Movement
Forecast whether a primary market and its causally visible sibling markets co-move over a daily horizon.

- **Release**: `t6.kdd.v2`; 766 train, 1,225 validation, and 2,592 test rows
- **Unit**: `(condition_id, bundle_day, horizon_days)` for 1, 3, and 7 days
- **Labels**: `no_effect`, `primary_only`, `cross_market`
- **Metrics**: Macro-F1 and accuracy overall/by horizon; four-way class and cascade-size metrics are analysis-only
- **Protocol**: Only prediction-time schema fields are model inputs; legacy intraday graph scripts are not v2 baselines

## Data

### Hosted snapshot

The hosted Hugging Face dataset was updated in August 2026 with the KDD v2
release: configs `t1`-`t6` hold the v2 files (T1 test 275 rows, T2
train/validation/test split, rebuilt T3 `train` plus the new human `gold`
split, T4 test 5,791 rows, T5 test 1,761 rows, T6 test 2,592 rows). `t7` is
a legacy alias. Access details are withheld for anonymous review.

```python
from datasets import load_dataset

ds = load_dataset("ANONYMOUS/EventXBench", "t1")  # repository id withheld for anonymous review
```

See [`data/README.md`](data/README.md) for the schema card and migration
status.

### Canonical July 2026 release data

A clean Git checkout contains the baseline code and documentation, not the
versioned KDD data directories. Obtain the dated
`*_latest_complete_bundle_20260726` release archives from the benchmark
release maintainers, verify the included manifest hashes, and pass the
unpacked directory explicitly, for example
`--local-dir KDD/data/t1_kdd_v2` or
`--data-dir KDD/data/t4_kdd_v2`.

T2 contextual baselines require auxiliary frozen artifacts that are not a
single loader split. Point `--data-dir` at a directory with this layout:

```text
/path/to/t2-release/data/t2/
|-- gold_r3_contextual_final/
|   |-- release_manifest.json
|   |-- val_candidates.jsonl
|   |-- val_labels.csv
|   |-- test_candidates.jsonl
|   `-- test_labels.csv
|-- t2_contextual_thresholds_q3_p2/
|   `-- FROZEN_THRESHOLDS.json
`-- train_silver_contextual_q3_p2/
    |-- train_labels_final.csv
    `-- generation_report.json
```

The T2 LLM runner additionally needs
`KDD/t2_recall_freeze_m3/frozen_candidates_retrospective.jsonl`; provide it
with `--train-candidates` when it is not under the repository root. Keep
test labels sealed for model selection and threshold tuning.

### Canonical Task Splits

| File / config | Description | Size |
|---------------|-------------|-----:|
| `t1` (`t1.kdd.v2`) | 709 train / 275 test markets | 984 |
| `t2` (`t2.gold.r3.contextual.v1`) | 544 train / 2,500 validation / 2,500 test | 5,544 |
| `t3` silver (`train`) | T3 evidence grades (0--5; `final_grade`) | 279,924 |
| `t3` human audit (`gold`) | T3 human-adjudicated grades (`gold_grade`) | 2,687 |
| `t4` (`t4.kdd.v2`) | 2,875 train / 2,268 validation / 5,791 test | 10,934 |
| `t5` (`t5.kdd.v2`) | 889 train / 692 validation / 1,761 test | 3,342 |
| `t6` (`t6.kdd.v2`) | 766 train / 1,225 validation / 2,592 test | 4,583 |

The 2,687-row T3 `gold` audit pool is sampled from the silver export, so its
count overlaps the 279,924-row `train` export and must not be added to it.

Versioned KDD directories also contain a `manifest.json` (and, where
applicable, a schema) that freezes counts, hashes, split policy, allowed input
features, and forbidden post-decision fields.

### Privacy

**Post-text access depends on the artifact.** The general post-ID corpus (`posts_no_text.jsonl`) omits tweet text and may require authorized rehydration (retrieving a post's content from its ID via the X API). T2/T3 task exports contain task-specific text fields under their applicable access conditions. Market-side questions, rules, price data, and the frozen T1 features can be used without rehydrating posts. Access and redistribution remain subject to source terms.

## Baselines

We provide three baseline families:

### LLM Baselines
Zero-shot and three-shot prompting through OpenAI, Anthropic, xAI, or an
OpenAI-compatible gateway:

```bash
# Inspect a T4 validation prompt without making an API request
python baselines/t4/llm_baseline.py \
  --provider openai --model MODEL \
  --data-dir KDD/data/t4_kdd_v2 \
  --split validation --shots 0 --dry-run

# Run a resumable validation job through an OpenAI-compatible gateway
python baselines/t6/llm_baseline.py \
  --provider openai --base-url https://HOST/v1 --model MODEL \
  --data-dir KDD/data/t6_kdd_v2 \
  --split validation --shots 3 --resume --output results/t6.validation.jsonl
```

Validation/development is the default workflow. Every LLM test run requires
the explicit `--allow-test` acknowledgement. Few-shot examples come only from
training/calibration data, temperature is zero, outputs are resumable, and
reports record prompt/configuration and data hashes. See
[`baselines/LLM_BASELINE_COMPATIBILITY.md`](baselines/LLM_BASELINE_COMPATIBILITY.md).

**API-key environment variables**: `OPENAI_API_KEY`,
`ANTHROPIC_API_KEY`, or `XAI_API_KEY`; gateways can select another variable
with `--api-key-env`.

### ML Baselines
LightGBM classifiers with Bayesian hyperparameter tuning (Optuna):

```bash
python -m baselines.t1.lightgbm_baseline --local-dir KDD/data/t1_kdd_v2
python -m baselines.t4.lightgbm_baseline --local-dir KDD/data/t4_kdd_v2
```

For the repeated market-day tasks (T4--T6), model-selection folds are
group-disjoint by `event_cluster_id`, with a row-wise `condition_id` fallback
when a cluster ID is unavailable. Frozen test rows are never used for tuning.

### Heuristic & Basic Baselines
Majority class, random walk, and frozen contextual retrieval:

```bash
python -m baselines.t1.basic_baseline --local-dir KDD/data/t1_kdd_v2
python -m baselines.t2.contextual_baselines \
  --data-dir /path/to/t2-release/data/t2 \
  --output-dir results/t2-contextual
python -m baselines.t6.basic_baseline --local-dir KDD/data/t6_kdd_v2
```

The T2 contextual runner preserves its CSV tables and report, and also writes
prediction-only JSONL plus a matching local gold JSONL for the unified
evaluator:

```bash
python evaluation/evaluate.py --task t2 \
  --predictions results/t2-contextual/bge_top1_frozen_threshold.test.predictions.jsonl \
  --gold results/t2-contextual/t2.test.gold.jsonl
```

## Evaluation

```bash
python evaluation/evaluate.py --task t1 \
  --predictions results/t1_predictions.jsonl \
  --gold KDD/data/t1_kdd_v2/test.jsonl
python evaluation/evaluate.py --task t4 \
  --predictions results/t4_predictions.jsonl \
  --gold KDD/data/t4_kdd_v2/test.jsonl
```

The evaluator requires explicit frozen gold by default. See
[`evaluation/README.md`](evaluation/README.md) for all-task directory naming,
prediction formats, and the explicit legacy `--hosted-gold` opt-in.

## Repository Structure

```
EventXBench/
|-- README.md
|-- LEADERBOARD.md
|-- requirements.txt
|-- eventxbench/                  # Data loading utilities
|   |-- __init__.py
|   `-- loader.py
|-- data/
|   `-- README.md                 # Dataset schema card
|-- baselines/
|   |-- t1/ ... t6/              # Canonical per-task baselines
|   |-- t7/                       # Legacy decay-only compatibility
|   `-- LLM_BASELINE_COMPATIBILITY.md
|-- evaluation/
|   |-- evaluate.py               # Unified evaluation CLI
|   |-- metrics.py                # Metric implementations
|   `-- README.md
|-- scripts/
|   `-- upload_to_hf.py           # Hosted-data publishing helper
`-- examples/
    `-- quickstart.py
```

## Leaderboard

See [LEADERBOARD.md](LEADERBOARD.md) for current results. To submit, open a pull request.

## License

- **Code**: MIT License
- **Data**: [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)
- Post text: see [Privacy](#privacy); access depends on the artifact
- Polymarket data are provided subject to the applicable platform API, data-use, and redistribution terms