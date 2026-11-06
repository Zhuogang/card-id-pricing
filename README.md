# card-id-pricing

A learning project that identifies English Pokémon trading cards from images and estimates their current reference prices using **free online data**.

Public card scans supply the catalog, downloaded real scenes supply evaluation images, and synthetic views support controlled experiments. No physical card collection or personal photography is required. Manual checks of public-image labels are allowed.

The pricing experiment hides every price field for selected cards and predicts their reference prices from metadata, images, and visible prices of other cards in the same dated snapshot. It studies price imputation; it does not yet forecast future sale prices.

## Status

**Planning stage.** The repository has documentation but no implemented pipeline, environment lockfile, or measured model results.

Read [the revised plan](PLAN.md) for the source validation, evaluation design, milestones, and completion criteria. It was revised on **2026-11-06** and is the authoritative plan. Follow [the implementation plan](IMPLEMENTATION_PLAN.md) for the twelve build steps, files, command interfaces, and acceptance checks. [The earlier guide](GUIDE.md) is retained as historical material and contains superseded assumptions.

## What the first version teaches

- Catalog identity, printing variants, data provenance, and reliable price joins.
- Image embeddings, exact nearest-neighbor retrieval, geometry, and synthetic-to-real differences.
- Candidate verification, confidence calibration, and abstention.
- Price baselines, gradient boosting, feature leakage, and prediction intervals.

Fine-tuning, forecasting, collection totals, approximate indexes, and hosted serving are extensions.

## Intended behavior

A selected card image produces candidate catalog identities. Accepted matches show prices separately for supported printings, with source, currency, and snapshot date. Ambiguous matches need review.

An available source quote is displayed as an observed reference price. In the hidden-price experiment, the model instead returns an estimate and interval. These outputs do not assess condition, authenticity, or the amount a particular physical card will sell for.

## Data

| Source | Planned use |
| --- | --- |
| [Pokémon TCG bulk catalog](https://github.com/PokemonTCG/pokemon-tcg-data) | Pinned initial card metadata and image references |
| [TCGCSV](https://tcgcsv.com/docs) | Product records and a frozen TCGplayer price snapshot |
| [TCG-AR public assets](https://github.com/ULiege-VIULab/tcg-ar/blob/main/docs/DATA.md) | Real scenes with card identities and corner annotations, after a full data audit |
| [TCGdex](https://tcgdex.dev/) | Alternative catalog/image source and supporting mapping evidence |
| Synthetic composites | Controlled image experiments and optional training |

The plan records which sources were actually probed and which checks remain. Downloaded assets will stay outside Git, with source manifests and applicable notices. The data-directory ignore rules still need to be added in M0.

## First steps

1. Create a minimal reproducible environment and record data sources.
2. Validate a small catalog-to-price pilot.
3. Freeze independent development, calibration, and test manifests.
4. Run identification and price baselines before adding complexity.

No setup command is provided yet because the package and dependency files have not been created.

## Results to report

| Experiment | Required evidence | Status |
| --- | --- | --- |
| Identification | Top-1, recall@k, gallery size, real versus synthetic results | Not run |
| Abstention | Accuracy among accepted queries, acceptance coverage, unknown false acceptance | Not run |
| Hidden-price estimation | Baseline comparison, log error, dollar error, price-tier counts | Not run |
| Price intervals | Measured coverage and width, with calibration assumptions | Not run |
| Integration | Separate observed quotes and estimates; effects of identification/mapping errors | Not run |

Every completed milestone will include saved predictions, a reproducible configuration, and a learning-log entry.
