# Implementation plan

**Status: proposed; no application code has been implemented.** Written on 2026-11-06.

[PLAN.md](PLAN.md) defines the learning goals, validated sources, and evaluation rules. This document turns those decisions into an ordered build backlog. If a scientific assumption changes, update the relevant rule in `PLAN.md` before changing the implementation.

The scope remains free online data, manual annotation of public images where useful, and estimation of a hidden card's current reference price. Build a small working experiment before expanding the catalog.

## 1. Implementation choices

These are defaults to implement, not claims that the environment has been tested.

| Area | Initial choice | Reason |
| --- | --- | --- |
| Runtime | Python 3.12; `uv`; `src/cardid/` package | One environment, package, and command entry point |
| Configuration | TOML loaded with `tomllib`; validated dataclasses | Explicit settings without another configuration dependency |
| Tabular work | pandas, NumPy, PyArrow; Parquet for tables | Keep the first pipeline easy to inspect |
| Downloads | httpx with caching, bounded retries, and pacing | Source responses remain reproducible and reusable |
| Images | Pillow; OpenCV added for geometry | Image decoding and card rectification |
| Retrieval | Frozen DINOv2; normalized vectors; NumPy exact search | Establish a baseline before training or indexing complexity |
| Pricing | Simple medians, then LightGBM on log price | One interpretable baseline family and one trained model |
| Calibration/metrics | scikit-learn plus small explicit metric functions | Keep statistical calculations inspectable |
| Demo | Local Gradio interface over ordinary Python functions | Reuse the evaluated pipeline |
| Development | pytest, Ruff, CPU checks with tiny generated fixtures | Test consequential data and evaluation errors |

Add dependencies when their step needs them. Lock the versions that actually install on this machine; select a compatible PyTorch build after checking its architecture and available GPU. Do not copy the historical guide's CUDA or package pins. The proposed `uv` workflow follows its [project documentation](https://docs.astral.sh/uv/guides/projects/).

Use files rather than a database or service. Paid APIs, hosting, scheduled collection, model fine-tuning, and full price-history backfills are outside this implementation.

## 2. Shared conventions

### Commands and configuration

Install a `cardid` console entry point backed by `argparse`. All command examples below are **interfaces to build**, not working instructions today. The common invocation will be:

```bash
uv run cardid --config configs/pilot.toml <command>
```

The configuration must name:

- Data, artifact, and result directories; random seed; schema version.
- Catalog commit, source manifest, pilot ID manifest, and source group list.
- Snapshot ID, crosswalk version, and identification/pricing split manifests.
- Model revision, preprocessing policy, and batch size.
- Selected pipeline stage, pricing feature set, and calibration artifact.

Freeze resolved values and file hashes with each run. Do not resolve “latest” inside a saved experiment. Optional dependencies must load only in the commands that need them; `doctor` and price ingestion must work without a GPU package.

### Files and artifacts

| Location | Contents | Git policy |
| --- | --- | --- |
| `configs/` | Small experiment settings | Commit |
| `manifests/` | Source records, pilot IDs, splits, mapping overrides, final experiment lock | Commit metadata where permitted |
| `data/raw/` | Original responses, reference images, downloaded real-image archive | Ignore |
| `data/derived/` | Catalog, crosswalk, snapshots, crops, synthetic views, features | Ignore |
| `runs/<run_id>/` | Models, embedding caches, predictions, logs, artifact metadata | Ignore |
| `results/` | Results index and reviewed reports/predictions permitted for publication | Commit selected artifacts |

Every derived artifact has metadata containing its schema version, input hashes, configuration hash, source revision, creation time, code revision, row count, and completion status. Record a patch hash for experiments run with local code changes. Write files to temporary paths and publish a completion marker only after validation. Failed jobs must not appear as complete runs.

Do not store public card artwork in test fixtures. Generate simple colored rectangles and tiny invented tables for offline tests.

### Boundaries that prevent leakage

| Component | Receives | Must not receive |
| --- | --- | --- |
| Identification inference | Image bytes, fixed gallery, model/configuration | Query's true ID, informative source filename, annotation labels |
| Price feature builder | Allowlisted metadata, frozen image vectors, visible fit-partition prices | Development/calibration/test price fields or a connection to raw provider data |
| Price predictor | Prepared features, fixed fitted model | Ground-truth target or an automatic lookup fallback |
| Calibrator | Fixed predictions and calibration labels only | Final-test labels |
| Evaluator | Predictions and the labels for the requested evaluation split | Permission to modify the predictor |
| Demo quote lookup | Explicit observed-quote mode and local snapshot | Access from the hidden-price prediction path |

Put label loading in evaluation/calibration modules. These boundaries prevent accidental leakage; they are not a security system against a person deliberately inspecting local files.

## 3. Ordered build backlog

Each step should leave a reviewable change and a runnable check. After each step, add a learning-log entry. Suggested command names may change before their first implementation; thereafter keep them consistent with the documentation.

| Step | Deliverable | Depends on | Learning milestone |
| --- | --- | --- | --- |
| 01 | Package, configuration, artifact helpers | — | M0 |
| 02 | Pinned catalog and reference images | 01 | M1 |
| 03 | Price snapshot and reviewed crosswalk | 02 | M1 |
| 04 | Public-image audit and frozen splits | 02, 03 | M2 |
| 05 | Evaluation harness and price visibility | 04 | M2, M6 |
| 06 | Frozen-encoder retrieval baseline | 05 | M3 |
| 07 | Price baselines and first fitted model | 05 | M6, M7 |
| 08 | Synthetic queries and rectification | 06 | M4 |
| 09 | Candidate verification and abstention | 08 | M5 |
| 10 | Optional neighbor features and price intervals | 07; 06 for image features | M7 |
| 11 | Local integrated demo | 09, 10 | M8 |
| 12 | Locked final evaluation and report | 11 | M8 |

Steps 06 and 07 can proceed independently once step 05 is complete. Do not make price modeling wait for image fine-tuning.

### Step 01 — Create the minimal project

**Files:** `pyproject.toml`, `.python-version`, `uv.lock`, `.gitignore`, `src/cardid/{__init__,__main__,cli,config,paths,artifacts}.py`, `configs/pilot.toml`, `LEARNING_LOG.md`.

- [ ] Add `/data/`, `/runs/`, local galleries, and checkpoint exclusions before downloading assets.
- [ ] Add the console entry point, strict configuration validation, and portable path resolution.
- [ ] Implement hashing, atomic writes, artifact metadata, and structured run logs.
- [ ] Add a `doctor` command reporting Python/platform, configured paths, and available optional packages.
- [ ] Lock the minimal environment. Add a CPU check workflow when there are meaningful tests to run.

**Accept:** package installation and `cardid --help` work; invalid configuration fails clearly; an interrupted artifact write cannot be mistaken for completion; `git check-ignore` confirms the planned data/run paths.

### Step 02 — Import the catalog and choose a pilot

**Files:** `src/cardid/catalog/{bulk,images,pilot}.py`, `src/cardid/data/http.py`, `manifests/sources.json`, `manifests/pilot_ids.csv`.

- [ ] Download or clone the bulk catalog at a recorded commit and normalize its metadata. Preserve source strings and IDs.
- [ ] Read the published TCG-AR annotation inventory to determine candidate benchmark IDs; the image audit happens in step 04.
- [ ] Reserve unknown-card groups before constructing the gallery. Build a pilot of about 1,000 catalog IDs from benchmark identities, metadata strata, and deliberate look-alikes.
- [ ] If those required identities exceed the pilot target, record a larger gallery rather than silently dropping hard cases.
- [ ] Download only selected reference images. Record hashes, dimensions, image mode, and decode failures; handle transparency explicitly.
- [ ] Produce a look-alike review list. Confirm shared artwork manually before using that label.

**Commands:** `catalog build`, `catalog images`.

**Outputs:** `catalog.parquet`, image manifest, pilot IDs, look-alike candidates/confirmed groups.

**Accept:** IDs are unique; collector numbers remain strings; reruns reuse valid cached downloads; every pilot reference either decodes or has an explicit exclusion reason. A synthetic fixture containing a prefixed number survives normalization unchanged.

### Step 03 — Capture prices and validate the joins

**Files:** `src/cardid/data/tcgcsv.py`, `src/cardid/catalog/match.py`, `manifests/set_map.csv`, `manifests/mapping_overrides.csv`.

- [ ] Retrieve the source update marker, needed product groups, and their price responses; check the marker again at the end.
- [ ] Retain raw responses. Mark a snapshot complete only when all configured groups pass schema checks and the update marker is consistent. Preserve failures without using a partial snapshot as complete.
- [ ] Use a custom User-Agent, conservative per-host pacing, bounded retry/backoff, and `Retry-After` where supplied. Read from local cache thereafter, following [TCGCSV's usage rules](https://tcgcsv.com/docs).
- [ ] Match verified set, collector number, name, language, and printing evidence. Filter sealed products and lots. Keep ambiguous matches unresolved.
- [ ] Review a stratified sample and every ambiguity; save manual decisions with evidence. Use optional TCGdex evidence only to assist review.
- [ ] Detect aliases sharing a marketplace product; exclude conflicting mappings or retain verified aliases under one identity group.

**Commands:** `prices snapshot`, `catalog match-prices`.

**Outputs:** immutable raw responses, `price_snapshot.parquet`, `crosswalk.parquet`, `crosswalk_review.csv`, coverage/exclusion report.

**Accept:** price-row keys are unique; accepted mappings have a single interpretation; null prices stay null; no condition/sales-volume field is invented; repeated ingestion produces no duplicate rows. An ambiguous fixture must produce an unresolved mapping rather than a guessed price.

### Step 04 — Audit public queries and freeze the experiments

**Files:** `src/cardid/data/real_queries.py`, `src/cardid/evaluate/splits.py`, versioned manifests under `manifests/`.

- [ ] Fetch the TCG-AR real-image archive and pinned annotation files; hash and decode every selected image.
- [ ] Confirm annotation flags, actual image dimensions, corner order, and card identity resolution. Record corrections separately from originals.
- [ ] Group duplicate/near-duplicate scenes and related captures before making crops. Review candidate duplicate groups manually.
- [ ] Freeze real-query partitions: approximately 40% development, 20% calibration, 40% final test by scene group.
- [ ] Freeze price partitions: approximately 60% fit, 15% development, 10% calibration, 15% final test by identity group, including every linked printing.
- [ ] Partition synthetic backgrounds/seeds and unknown-card groups. Save the allocation before generating queries.
- [ ] Reconcile any catalog exclusions from the audit, then freeze the final pilot gallery and all manifests together.

If there are too few independent real groups, use the real dataset only for final evaluation and develop on synthetic or separately annotated public data, as specified in `PLAN.md`. Do not claim real-photo calibration without a real calibration partition.

**Commands:** `datasets prepare`, `datasets audit`, `datasets freeze`.

**Accept:** no related scene crosses real-image partitions; no price identity/product alias crosses price partitions; every eligible query resolves to the gallery or a declared unknown class. All changes after freezing require a new manifest version.

### Step 05 — Build the evaluator and visible-price data layer

**Files:** `src/cardid/pricing/{visibility,features}.py`, `src/cardid/evaluate/{identify,pricing,metrics}.py`, `src/cardid/results.py`, focused tests.

- [ ] Materialize a fit-only price table and separate label tables. Feature builders accept the former, never the full snapshot.
- [ ] Implement group-aware out-of-fold construction of any price-derived features for fit rows. Use the complete fit partition to compute features for development/calibration/test queries.
- [ ] Strip source IDs from model inputs where they reveal the label; feed image bytes under opaque query IDs.
- [ ] Implement top-1/recall@k, per-card summaries, selective accuracy, rejection metrics, log/dollar errors, interval coverage, and paired grouped bootstrap summaries.
- [ ] Count abstentions, missing predictions, exclusions, and empty slices explicitly. Missing predictions must not disappear from denominators.
- [ ] Save per-query/per-price predictions before aggregating results. Register runs with data/configuration/code hashes.

**Commands:** `evaluate identify`, `evaluate prices`; development split is the default.

**Accept:** hand-computed fixtures match expected metrics. Changing every hidden target leaves all model features unchanged. Adding the target's sibling printing or alias to the candidate neighbor table cannot expose its hidden price. In each fit fold, an identity cannot contribute its own target-derived features.

### Step 06 — Establish retrieval baselines

**Files:** `src/cardid/identify/{preprocess,embed,search}.py`.

- [ ] Implement perceptual hashing as a simple comparison and one frozen DINOv2 encoder.
- [ ] Cache reference vectors with ordered card IDs and model/preprocessing/gallery hashes.
- [ ] Use deterministic preprocessing and normalized-vector exact search. Start with the model's documented processing; compare an explicit full-card padding policy on development data if cropping removes identifying text.
- [ ] Run raw selected crops and annotation-corner crops separately. Never use true query IDs to reduce the candidate gallery.
- [ ] Measure whole-query latency, including decoding and preprocessing, on the actual hardware.

**Commands:** `identify index`, `identify run --split dev --stage raw`, `identify run --split dev --stage oracle`.

**Accept:** every query searches the same frozen gallery; rankings are stable for fixed inputs; a small exact-search fixture matches a brute-force calculation. A cached index with the wrong gallery/preprocessing hash is rejected. Record development metrics and failure examples even if performance is poor.

### Step 07 — Build price baselines and one trained model

**Files:** `src/cardid/pricing/{baselines,model}.py`; extend `features.py`.

- [ ] Implement the four baselines from `PLAN.md`: printing/global median, set/rarity/printing median, same-Pokémon peers, and verified same-art peers.
- [ ] Implement deterministic fallback order and record which fallback each prediction used.
- [ ] Build an allowlisted metadata feature matrix. Fit categorical mappings on fit data and provide explicit unknown/missing categories.
- [ ] Train LightGBM with absolute-error loss on log price; exponentiate predictions to dollars.
- [ ] Predeclare a small development tuning budget, such as at most six configurations. Do not make a large parameter search part of the first version.
- [ ] Select between the fitted model and the best baseline using development results; keep the fit partition unchanged.

**Commands:** `pricing baseline --split dev`, `pricing train`, `pricing predict --split dev`.

**Accept:** all methods cover the same eligible development cohort, with fallback and failure rates reported. Held-out IDs never enter fitting. Unseen categorical values work, predictions are positive/finite, and model/feature hashes are stored. A baseline win is a valid outcome.

### Step 08 — Generate synthetic queries and rectify cards

**Files:** `src/cardid/synth/generate.py`, `src/cardid/identify/rectify.py`.

- [ ] Generate single-card composites with exact corner labels and independent split-specific seeds/backgrounds.
- [ ] Start with procedural backgrounds, perspective, rotation, blur, scaling, noise, and JPEG effects; add downloaded textures/glare only after this works.
- [ ] Implement the contour/quad rectifier; reject degenerate transformations and retain the raw crop on failure.
- [ ] Compare raw, automatic, and annotation-corner crops on the same development queries.
- [ ] Report corner error, rectification failures, identification changes, and performance by synthetic corruption type.

**Commands:** `synth generate --split dev`, `identify run --split dev --stage auto`.

**Accept:** known synthetic corners survive the transform/inverse-transform check; degenerate quads return an explicit failure; every query remains represented in downstream metrics. Real-image and synthetic-image result rows are separate.

### Step 09 — Add one verification method and abstention

**Files:** `src/cardid/identify/{verify,confidence}.py`.

- [ ] Choose local feature matching or OCR based on development failures; implement one first.
- [ ] Restrict reranking to retrieved candidates and record candidate-set recall. Keep missing or unreliable verification evidence neutral.
- [ ] If learning a score combination, fit it on development data with scene-aware cross-validation.
- [ ] Freeze retrieval, preprocessing, and verification before calibration.
- [ ] Fit a simple confidence calibrator on the calibration split and choose an acceptance threshold using a criterion recorded before inspecting calibration outcomes.
- [ ] Store the calibrator and threshold with all pipeline hashes. If calibration has too little variation or no real data, return a labeled similarity score instead of a fitted probability.

**Commands:** `identify verify --split dev`, `identify calibrate`.

**Accept:** confidence artifacts refuse mismatched pipelines. False acceptance of unknowns and acceptance counts are measurable. Threshold failure can produce low coverage or full abstention; it must not trigger final-test tuning.

### Step 10 — Calibrate price ranges

**Files:** `src/cardid/pricing/intervals.py`; optional `neighbors.py`.

- [ ] If useful, add fit-only image-neighbor price features and compare them on development data before choosing the final predictor.
- [ ] Reuse the visibility/out-of-fold rules from step 05. Image features do not justify exposing any held-out price.
- [ ] Freeze the selected predictor and feature pipeline. Predict calibration prices in log space.
- [ ] Compute one score per calibration identity group: the maximum absolute log residual across its modeled printings.
- [ ] Sort scores; use the one-based rank `ceil((n + 1) * 0.8)`. If the rank exceeds `n`, mark the interval unsupported.
- [ ] Save `q`, group count, target coverage, and model/split hashes. Return `exp(log_prediction - q)` and `exp(log_prediction + q)`.

**Command:** `pricing calibrate`.

**Accept:** a small hand-computed example verifies the quantile and group maximum; endpoints are ordered and positive; calibration never fits the predictor or reads test labels. Replacing a model invalidates its interval artifact. The report distinguishes per-printing coverage from simultaneous group coverage and states the exchangeability assumptions from `PLAN.md`.

Do not use final coverage to choose between interval methods. Quantile models are a later development experiment requiring a fresh calibration procedure.

### Step 11 — Connect a local demo to the evaluated functions

**Files:** `src/cardid/app/{service,ui}.py`.

Keep the service layer callable without Gradio. Use these proposed result contracts:

| Result | Required fields |
| --- | --- |
| Identification | `status, candidates, score_kind, score, accepted_card_id, gallery_version, pipeline_id` |
| Price | `status, card_id, source_printing, mode, amount_usd, interval_usd, source, snapshot_id, model_id` |

- [ ] Implement `identify(image_bytes)`, `list_printings(card_id)`, and `price(card_id, printing, mode)`.
- [ ] Make `mode` explicit: `observed` reads a stored quote; `estimated` uses the fixed model and visible-fit features.
- [ ] Choose hidden-price demo examples from identities that were not fitted as targets, or genuinely unpriced identities. Select examples using their manifest, then pass only image bytes to inference. Do not filter an evaluation query out because its predicted identity is wrong or belongs to the fit partition; that remains an end-to-end error or abstention.
- [ ] Return clear statuses for review, unsupported printing, unresolved mapping, absent quote, and unavailable estimate.
- [ ] Use development/fit examples while building the UI. Keep final-test queries and labels out of example galleries before the final evaluation.
- [ ] Bind locally and load artifacts once. The interface performs no provider calls.

**Command:** `demo`.

**Accept:** the estimated path cannot call the quote lookup; ambiguous printing is never silently selected; abstention suppresses automatic pricing. Tests cover these branches with stubbed models and tiny tables. The same preprocessing/model functions power both evaluation and the demo.

### Step 12 — Lock, evaluate, and write the report

**Files:** `src/cardid/evaluate/{final,report}.py`, `manifests/final.toml`, `results/`, README updates.

- [ ] Record the exact baseline/final configurations, gallery, splits, snapshots, fitted artifacts, thresholds, metrics, and intended comparison before running final-test predictions.
- [ ] Have `evaluate final` verify the lock and record every invocation. Standard development commands should refuse `split=test` and direct the user to this command.
- [ ] Run the planned comparisons in one final batch. Record partial execution if a job fails. Reuse valid cached predictions when resuming unchanged jobs.
- [ ] Report identification and pricing independently. For combined hidden-price metrics, use only their eligible test intersection with verified printing labels; count all abstentions and exclusions.
- [ ] Report baseline wins, unsupported slices, missing sources, and wide intervals without changing the selection.
- [ ] Add two reproduction paths: metrics from saved predictions, and inference from cached/obtainable pinned inputs. Report any part that cannot be reproduced from currently available public sources.

**Commands:** `experiment lock`, `evaluate final --lock manifests/final.toml`, `reproduce --from predictions`, `reproduce --from inputs`.

**Accept:** rebuilding metrics reproduces saved values; inference reproduces them within a documented numerical tolerance; no results are presented without their data/split versions. A test-informed change is exploratory or uses a new untouched test version.

## 4. First implementation checkpoint

Build **steps 01–05 first**, with small downloads and fixtures. This is the first review checkpoint, not a request for additional permission.

It must produce:

1. One runnable package and locked environment.
2. A pinned catalog pilot, validated local images, one dated price snapshot, and a reviewed crosswalk.
3. Versioned image/price splits with leakage checks.
4. A dummy prediction file that the metric harness evaluates correctly.
5. A short learning-log entry describing data issues and decisions.

Then implement steps 06 and 07 to get the first meaningful identification and pricing results. Expand the gallery, add geometry, or improve the models only after these baselines are available.

## 5. Definition of done for each change

Before marking a step complete:

- Its artifact or command exists and the documented acceptance check passes.
- The run records its exact inputs and configuration.
- Consequential data/leakage cases have meaningful tests; documentation and straightforward plumbing do not need tests that merely repeat their implementation.
- Development results and limitations are recorded without using final-test outcomes for choices.
- The learning log explains the concept learned and the next experiment.

This document is the build backlog. The full first-version completion criteria remain in [PLAN.md](PLAN.md#9-completion-criteria).
