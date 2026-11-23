> **Code repository (with the core bond-completion checkpoints).** The full trained model set, the datasets and the production result tables are distributed as release assets (see *Releases*), not in this git repository.

> How to run: enter the package root (`SCULPTOR_v1.0.0`), run
> `powershell -ExecutionPolicy Bypass -File .\setup_env.ps1`, then invoke the
> CLI with `.\.venv\Scripts\python.exe -m fuel_design.cli ...`.
> The full distribution is assembled from this repository plus the release assets (models and data); `setup_env.ps1` rebuilds the Python environment.

# Symmetric Fuel Design

This package implements the SCULPTOR workflow described in the accompanying manuscript: a Python toolkit for
symmetry-constrained fuel molecule generation. The project supports two
generation lines:

1. **Symmetric molecule line**: base graph -> semi-base graph (stage-1 GNN) ->
   complete half graph (stage-2 GNN) -> mirror completion -> full symmetric
   molecule.
2. **Perturbation line**: starting from the Tier 1-3 symmetric molecules, apply
   local carbon-skeleton perturbations so the products no longer need to be
   perfectly symmetric.

Both lines share the same post-processing: SMILES normalization and dedup,
RDKit/MMFF screening, property prediction (density, heating value, melting
point, boiling point, flash point), e3nn geometry/symmetry scoring, SA score,
novelty scoring, composite ranking, and the two-axis channel split (Tier 1-4 for
threshold-passing molecules, Potential 1-3 for the rest).

## Molecular representation

Every symmetric saturated hydrocarbon is compressed into a half graph encoded
as `(n1, n2, n3, edges)`:

- `n1`: carbons on the mirror axis (fixed points).
- `n2`: carbons that swap with their mirror partner AND have a cross-axis bond
  between the two partners.
- `n3`: ordinary mirror-paired carbons without a cross-axis bond.

`n2` and `n3` are full-molecule counts, so the half graph stores `n2/2` and
`n3/2` representative nodes.

## Compute device (`--device-mode`)

Only the property heads can use a GPU; generation, screening, perturbation and
the analysis steps always run on the CPU. Three explicit modes keep that choice
in the hands of the person reproducing a run:

| mode | behaviour |
|------|-----------|
| `cpu` | never touches CUDA - the whole workflow runs on the CPU |
| `gpu` | uses the GPU for the property heads; stops with an explanatory error if CUDA cannot actually be used |
| `auto` (default) | probes CUDA with a short smoke test and takes the GPU path only when the probe succeeds, otherwise the CPU path |

The switch is accepted before or after the sub-command, and the control panel
exposes the same three options:

```powershell
.\.venv\Scripts\python.exe -m fuel_design.cli --device-mode cpu  run-pipeline --checkpoint ... 
.\.venv\Scripts\python.exe -m fuel_design.cli run-pipeline --device-mode auto --checkpoint ...
```

Child processes inherit the choice through the `SCULPTOR_DEVICE_MODE`
environment variable, so a run pinned to `cpu` stays on the CPU end to end.

### The `auto` probe runs once, not once per call

The smoke test is executed at the first device lookup of a process and the
result is cached for the rest of that process. The resolved device is also
written to `SCULPTOR_RESOLVED_DEVICE`, which every child process inherits, so
worker processes and follow-up commands reuse the decision instead of probing
CUDA again. A production run therefore probes once per process tree, and the
control panel resolves `auto` once per job before it launches any command.

Log lines: exactly one `[device] mode=... -> ...` line is printed per process;
the first process states the smoke-test result and the descendants report
`skipped (inherited)`.

`SCULPTOR_DEVICE_NO_CACHE=1` restores the previous behaviour (probe on every
lookup). It exists only for measuring the difference - leave it unset in a run.

Indexing rules (IDs come from the full molecule, so they can be discontinuous):

- `n1`: `0 .. n1-1`
- `n2` originals: `n1 .. n1+n2/2-1`; mirrors: original `+ n2/2`
- `n3` originals: `n1+n2 .. n1+n2+n3/2-1`; mirrors: original `+ n3/2`

Degree rules:

- every carbon in the completed molecule has maximum degree `4`;
- `n2` representative carbons have maximum degree `3` in the half graph because
  mirror completion adds the fourth cross-face bond;
- an `n1` node's completed degree is computed from the half graph as
  `n1-n1_count + 2 * n1-n2_count + 2 * n1-n3_count`, and must be at most `4`;
- the `n1`-induced subgraph must be a **linear forest**: acyclic and with every
  `n1` node having at most 2 `n1` neighbors. There is **no adjacency-index
  restriction** on `n1`-`n1` bonds; the rule is purely topological.

Graph levels:

- **Base graph**: built by sequentially attaching n3 atoms to main structure
  nodes (`n1` nodes plus `n2` originals); it is an **acyclic forest**, every
  connected component contains **exactly one** main structure node, and no
  main-structure-to-main-structure edges are added.
- **Semi-base graph**: a base graph plus stage-1 connection edges; it has
  exactly one connected component (and, because it only connects different
  forest components, it is still acyclic).
- **Complete half graph**: the half-side topology after all legal bonds are
  restored; it may contain rings and does not include n2 cross-axis bonds.
- **Full molecule graph**: obtained by deterministic mirror completion of the
  complete half graph (mirror edge copying plus n2 cross-axis bonds), then
  checked for connectivity, self loops, duplicate edges, and degree limits.

Mirror completion rules:

1. keep all half-graph edges;
2. for every n2 original `i`, add the cross-axis edge `(i, i + n2/2)`;
3. mirror-copy every edge that touches at least one moving node (n1-n1 edges
   are not copied).

The pure graph layer has no third-party dependencies, so it can be tested before
installing chemistry or ML packages.

## Pipeline overview

```text
base graph -> stage-1 GNN (base -> semi-base) -> semi-base graph
           -> stage-2 GNN (semi-base -> complete half) -> complete half graph
           -> mirror completion -> RDKit sanitize + canonical SMILES
           -> exact dedup + fingerprint novelty -> MMFF geometry screening
           -> property prediction (density / heating value / melting / boiling / flash)
           -> e3nn geometry & symmetry scores + SA score
           -> composite ranking -> Tier 1-4 / Potential 1-3 channel tables
           -> perturbation seeds (the Tier 1-3 pools)
           -> perturbation generation -> same screening & routing
           -> manual review pool (symmetric + perturbed candidates)
```

## Quick checks

```powershell
cd .
$env:PYTHONPATH=(Get-Location).Path
.\.venv\Scripts\python.exe -m fuel_design.cli generate-base --seed 7
.\.venv\Scripts\python.exe -m fuel_design.cli count-partitions --total-carbons 6
.\.venv\Scripts\python.exe -m fuel_design.cli mirror --n1 2 --n2 2 --n3 2 --edge 0,2 --edge 1,4 --edge 2,4

# Training/data-prep commands use the datasets included under dataset/

# Generation / evaluation / end-to-end pipeline
.\.venv\Scripts\python.exe -m fuel_design.cli predict-bonds --checkpoint Generation\bond_completion_2\model\grouped_20260925\stage2_bond_completion_grouped_seed2026.pt --n1 2 --n2 2 --n3 4 --edge 0,5 --edge 4,5 --threshold 0.94 --top-k 3
.\.venv\Scripts\python.exe -m fuel_design.cli run-pipeline --checkpoint Generation\bond_completion_2\model\grouped_20260925\stage2_bond_completion_grouped_seed2026.pt --semi-base-checkpoint Generation\bond_completion_1\model\grouped_20260925\stage1_semi_base_grouped_seed2026.pt --semi-base-threshold 0.73 --known-smiles Generation\screening\data\known_smiles_total_reference.txt --total-carbons 13 --samples 100 --iterations 1 --threshold 0.94 --top-k 2 --seed 54 --output-dir result\pipeline_run

# Perturbation line (input tables are under result\prod_full_2026_10_02)
.\.venv\Scripts\python.exe -m fuel_design.cli run-perturbation-pipeline --finalized-hits result\prod_full_2026_10_02\perturbation_seeds_T123.csv --output-dir result\perturbation_run --samples 10 --known-smiles Generation\screening\data\known_smiles_total_reference.txt --temporary-smiles result\prod_full_2026_10_02\generation_temporary_smiles.txt --seed 16 --novelty-threshold 0.75 --samples-per-seed-operation --e3nn-geometry-checkpoint Evaluation_Iteration\evaluation\model\e3nn\e3nn_geometry_scorer_g_new.pt

# High-level calculation inputs for a chosen pool (Tier 1 by default)
.\.venv\Scripts\python.exe -m fuel_design.cli export-candidate-inputs --csv result\prod_full_2026_10_02\analysis\totals\tier1_candidates_total.csv --output-dir result\dft_inputs --smiles-column canonical_smiles --id-column candidate_id
```

## Generation modes

- `generate-base-by-counts` builds exploratory base graphs for a fixed
  `(n1, n2, n3)` by attaching n3 atoms into anchored n3 groups.
- `generate-semi-base-by-counts` first builds a base graph, then adds legal
  cross-component bonds until the representative half graph has exactly one
  connected component.
- `generate-bases-for-total` enumerates all `(n1, n2, n3)` partitions for a
  fixed total carbon count, then generates bases independently for each
  partition (skipping partitions without any main structure node).
- `preprocess-gdb` builds supervised training samples by deleting bonds from
  known GDB complete half graphs. With semi-base samples enabled (the current
  default flow) it also connects each base graph into a semi-base graph using
  only target edges, so the training CSV contains both stage-1 labels
  (`semi_base_edges_added`) and stage-2 labels (`candidate_edges` /
  `candidate_labels`).

For each molecule, `preprocess-gdb` keeps only distinct base variants.
Deduplication uses `base_canonical_key`, a canonical graph key under
type-preserving relabeling; `base_signature` is the human-readable coarse check
for n3 groups and their n1/n2 attachments. The semi-base extension key combines
the base key with a canonical code of the added connection edges.
`--variant-attempts` controls how many random deletion attempts are allowed
while searching for the requested number of unique variants.

The full preprocessed CSV is an audit table; use `export-training-csv` (stage-2)
or `export-semi-base-training-csv` (stage-1) before training to write compact
training tables.

## Bond-completion models (two independent GNNs)

The bond-completion step uses two structurally identical but independently
trained graph neural networks:

1. **Stage 1 (base -> semi-base)**: `predict_semi_base` scores only candidate
   edges whose endpoints lie in different connected components of the current
   graph. It adds one edge per step and recomputes components until the graph is
   connected. If no candidate reaches the threshold, the top-scoring legal edge
   is chosen so the pipeline never stalls.
2. **Stage 2 (semi-base -> complete half)**: `predict_bonds` scores every legal
   missing edge of the input graph and selects edges by threshold, temperature,
   and optional top-k. Legality is enforced at prediction time: degree limits,
   n1 linear forest, no self loops/duplicates, and the edge must survive mirror
   completion validation.

Architecture (shared by both stages):

- 10-dimensional node features: n1/n2/n3 one-hot, main-structure flag,
  normalized half degree, normalized remaining valence (n1 uses its mirrored
  full degree), normalized node id, and `n1/N`, `n2/N`, `n3/N`;
- `hidden_dim`-wide embedding, then `num_layers` of mean-aggregation message
  passing (`self_linear + neighbor_linear`, ReLU, LayerNorm) with dropout;
- each candidate edge `(u, v)` is scored from
  `[h_u, h_v, |h_u - h_v|, h_u * h_v]` through an MLP to a single logit;
- loss is binary cross-entropy with a positive-class weight derived from the
  training positive/negative ratio.

Current full C13 checkpoints (trained on `13.g.smi`, 40 epochs, batch 256,
`hidden_dim=128`, `num_layers=4`, CUDA):

- `Generation/bond_completion_1/model/semi_base_gdb13_c13_full.pt`:
  validation accuracy 0.8066, precision 0.541, recall 0.760, F1 0.632 at 0.5;
  the automatically calibrated recommended threshold is **0.67** (best F1
  0.665).
- `Generation/bond_completion_2/model/bond_completion_gdb13_c13_full_semibase.pt`:
  validation accuracy 0.9137, precision 0.287, recall 0.945, F1 0.440 at 0.5;
  positive label fraction 3.75%; the recommended threshold is **0.94** (best
  F1 0.652). Use it as the initial stage-2 threshold; closed-loop feedback can adjust
  it round by round.

Older `Generation/bond_completion_2/model/bond_completion_min6_to13_semibase*.pt` checkpoints remain for
regression tests and small-scale experiments only.

Reviewer-audit retraining (2026-09-25) uses a molecule-level split keyed by
`canonical_smiles`, so variants of one target molecule cannot appear in both
training and validation. The non-overlapping checkpoints are stored under
`Generation/bond_completion_1/model/grouped_20260925/` and
`Generation/bond_completion_2/model/grouped_20260925/`. With seed 2026, the
stage-1 validation set contains 1,687 molecules and gives a calibrated F1 of
0.6548 at threshold 0.73; the stage-2 validation set contains 1,702 molecules
and gives a calibrated F1 of 0.6314 at threshold 0.94. The accompanying
`.split.json` and `.history.json` files preserve the exact split and epoch
history. Metrics above this paragraph belong to the historical row-level split
and should only be used to reproduce older runs.

## Screening

`screen_generation` (and the equivalent logic in the perturbation line)
computes RDKit descriptors and applies soft hydrocarbon rules:

- connected and novel (not in the known-SMILES library);
- H/C ratio in `[0.7, 2.2]`, ring count in `[1, 6]`, rotatable bonds `<= 4`,
  fraction sp3 `>= 0.95`, SA score `<= 4.5`;
- MMFF embedding + optimization must succeed, with a minimum stability score
  and a maximum energy per heavy atom;
- total ring count is the full-molecule cycle rank. For this toolkit's neutral,
  connected, saturated hydrocarbons it is obtained exactly from the molecular
  formula as `R = C + 1 - H/2`; the completed graph is used as an invariant
  check. RDKit's SSSR count is stored separately as `rdkit_ring_count`, and
  candidate rows record
  `ring_count_definition=formula_cycle_rank_full_molecule_v1`;
- five- and six-member rings receive a small preference reward, while three- and
  four-member rings contribute a strain penalty. Both use the completed
  molecule's ring information, not the half graph;
- the half-graph projected counters remain diagnostic columns only. They never
  gate screening or contribute to candidate ranking;
- the trained property heads keep their existing internal ring descriptors;
  this pre-release correction applies to candidate screening, ranking, routing,
  and the perturbation-iteration input table;
- novelty = `1 - max Tanimoto similarity` against the known library using
  Morgan radius-4, 8192-bit fingerprints (upgraded 2026-08-06; radius-2/2048
  had same-fingerprint collisions among saturated hydrocarbon skeletons);
  `high_novelty` is `>= 0.75` by default (calibrate with
  `calibrate-novelty-threshold`).

## Property prediction and routing

The property layer consists of **six properties**. Five of them have prediction
heads; the mass heating value is **computed**, not predicted, and therefore has
no prediction head:

| # | Property | Prediction head | Implementation |
| --- | --- | --- | --- |
| 1 | density | yes | pure STA (token-additive SMILES transformer, `Evaluation_Iteration/evaluation/model/STA/sta_820k_v2/sta_gdb_density_paper.pt`); the thesis vdW method is available as an alternative |
| 2 | formation enthalpy | yes | pure STA (`Evaluation_Iteration/evaluation/model/STA/sta_820k_v2/sta_gdb_formation_enthalpy_paper.pt`); Yang CBH-BDC delta-ML is available as an alternative |
| 3 | mass heating value | **no** | computed from formation enthalpy through the Hess-law combustion cycle (`mass_heating_value_from_formation_enthalpy`); volumetric heating value = `density * mass_heating_value` |
| 4 | melting point | yes | phase-fusion regressor (default schema2 = 2D; schema3 additionally includes e3nn 3D features) |
| 5 | boiling point | yes | phase-fusion regressor (default schema2 = 2D; schema3 additionally includes e3nn 3D features) |
| 6 | flash point | yes | phase-fusion regressor (default schema2 = 2D; schema3 additionally includes e3nn 3D features) |

The control panel (`tools/control_panel`) exposes exactly this selection:
density (`cheng_vdw_density` / `sta_density` / `none`), formation enthalpy
(`yang_cbh_bdc` / `sta_formation` / `none`), heating value
(`yang_cbh_bdc` / `sta_heating` / `none`, computed either way), melting point
(`sfs_vr_reference` / `sta_melting` / `phase_fusion_melting` / `none`), boiling
and flash points (`none` / `sta_*` / `phase_fusion_*`), plus molar volume
(derived) and ISP (STA, optional). Molar volume is always derived as
`M / density` with propagated uncertainty and is an input to the phase-fusion
models; molecular weight is computed from the formula and is reference-only.

STA paper checkpoints (`Evaluation_Iteration/evaluation/model/STA/sta_820k_v2/sta_gdb_*_paper.pt`) are
token-additive SMILES transformers (d_model 512, 8 heads, 12 layers) trained on
the total deduplication library (`Evaluation_Iteration/evaluation/data/gdb820k_with_experimental.csv`);
each prediction carries its validation-MAE uncertainty. Molar-volume values are
converted from the source L/mol scale to public cm3/mol.

Phase-fusion (STA + e3nn) details: each of melting / boiling / flash is an
independent regressor whose auxiliary inputs are the STA-predicted values
(density, molar volume, formation enthalpy, ISP and the other phase points as
available) plus 2D structural descriptors; schema 4 additionally feeds a shared
30-dim e3nn 3D feature vector computed from a fixed-seed RDKit conformer. The
melting-point prediction is reference-only by default (value + uncertainty
interval); boiling and flash only participate in routing when explicitly
enabled. The SFS-VR melting model (thesis Chapter 3 voting model) remains
available as the reference-only melting option.

The e3nn geometry/symmetry scoring is separate from the property heads: it
scores conformer quality (geometry score, symmetry consistency, MMFF stability,
compactness, reproducibility), with an optional trained scorer checkpoint
(`Evaluation_Iteration/evaluation/model/e3nn/e3nn_geometry_scorer_g_new.pt`) and a rule-based fallback.

Routing uses absolute thresholds with uncertainty margins:
- density `>= 0.85 g/cm3`;
- heating value `>= 38 MJ/kg` (the consensus route also accepts `37-38 MJ/kg`
  as a no-false-reject floor via `--heating-value-possible-min`);
- SA `<= 4.5` (hard pre-screen);
- melting / boiling / flash only participate when explicitly enabled; melting is
  reference-only by default.

A prediction that clearly fails a threshold is rejected; a prediction that can
still cross the threshold within its uncertainty margin is marked for review;
fully conforming candidates pass. Missing selected properties write to the
missing-property table with a penalty (0.15 per property); a molecule whose
required heating value is unavailable is such a data gap and carries no channel
until the value is supplied (for example through `--heating-values-csv`).

Two-axis channel split. A molecule that meets **every** enabled hard threshold
lands on the tier axis, split by how many of the three compared properties beat
JP-10: Tier 1 (all three), Tier 2 (two), Tier 3 (one), Tier 4 (none). A molecule
that misses a hard threshold lands on the potential axis, where every pool is
named by how many thresholds are missed and how many properties still beat JP-10:
Potential 1 (1 missed, 2 beaten), Potential 2 (1 missed, 1 beaten), Potential 3
(2 missed, 1 beaten). A molecule that misses thresholds and beats JP-10 on
nothing matches no pool and keeps an empty channel in the property table. Every run writes
one table per channel (`tier1..tier4_candidates.csv`, `potential1..3_candidates.csv`),
`analysis/totals/` carries the merged version of all seven plus
`tier_summary.csv` with the per-stage counts, and `retier-run-outputs` re-derives
the split of an existing folder from its stored predictions (no model call). The
channels are pools; which molecule goes to a high-level calculation is a manual
choice made from them (`export-candidate-inputs --csv <pool>` writes the input
folder for any selection).

Composite ranking score (top-level weights must sum to 1):
- `property_quality` 0.65 (includes SA and structure validity),
- `e3nn_geometry` 0.10,
- `e3nn_symmetry` 0.10,
- `novelty` 0.15.

Property sub-scores are piecewise-linear around the hard thresholds: threshold
value scores 0.5, 15% better scores 1.0, 15% worse scores 0.0. SA maps to
`[0,1]` with `<=1.5 -> 1`, `3.0 -> 0.5`, `>=4.5 -> 0`.

## Iterative feedback

When `--feedback-reward` is enabled, each round:

1. generates and screens molecules;
2. predicts properties immediately for pre-screen-passing molecules
   (unless `--disable-online-property-feedback` is used);
3. computes the structural reward and a shaped property objective
   (composite score reshaped around 0.65 with slope 2.5, blended with the
   screening reward: pass 1.0 / review 0.45 / reject 0.0 / missing 0.1);
4. halves the reward of repeated SMILES;
5. updates the stage-2 threshold, temperature, top-k, the `(n1,n2,n3)`
   partition sampling weights, and per-edge logit biases
   (edge key = endpoint kind + degree + shortest-path distance);
6. keeps 12% uniform exploration (up to 35% when the repeat rate rises).

The first 20 rounds use a warmup property weight of 0.35; afterwards the
property weight is 0.70. Low-novelty e3nn-queue candidates additionally produce
weak STA-based iteration labels
(`export-sta-weak-feedback`, weight 0.35 scaled by novelty reliability). Closed-loop
feedback is a policy/strategy update; it does **not** back-propagate into the
GNN weights each round.

## Phase fusion models (melting / boiling / flash)

Three independent regressors per head, trained on the total deduplication library
(812,028 rows). The target table was modelized upstream, but no predicted
property is fed back as an input descriptor to either head.

- **schema2_mlp** (reported-run and reproducibility default,
  `Evaluation_Iteration/evaluation/model/schema 2/phase_fusion_gdb820k_mlp_ens2d/`):
  deep MLP ensemble (2 seeds, 45 structural descriptors, **no predicted property
  inputs**); this is the head used for the strict run reported in the manuscript.
- **schema3_mlp** (optional 3D descriptor variant,
  `Evaluation_Iteration/evaluation/model/schema 3/phase_fusion_gdb820k_mlp_ens3d/`):
  deep MLP ensemble (2 seeds, 45 structural/geometric + 30 e3nn spherical-harmonic
  invariant dims, **no predicted property inputs**, so no property back-substitution);
  649k train / 81k validation / 81k test rows; test MAE 1.546 K (melting),
  1.107 K (boiling), 1.948 K (flash), R虏 0.983/0.998/0.993; 90% conformal
  half-widths 3.76/2.70/4.72 K.
- The legacy tree heads (`schema2`, `schema3`, plus their `_repr` variants) were
  retired on 2026-09-22 and are no longer part of the runtime layout.

## Perturbation line

- A run folder carries two seed tables and the panel offers both. `build-t123-seeds`
  writes `perturbation_seeds_T123.csv` from the Tier 1-3 candidate pools (candidates
  that match or beat JP-10 on the hard-screened properties); `build-potential-seeds`
  writes `perturbation_seeds_potential.csv` from the potential pools - every
  Potential 1 molecule plus the best 10% of Potential 2 and of Potential 3 by
  composite score. The
  generation chain writes both, and the perturbation chain refreshes the potential
  table after each campaign because the pools grow with it. Perturbation does not
  wait for real quantum-chemical calculations: the seed tables are the pools the
  user chose to explore, and every seed row records the pool it came from.
- Seven perturbation operators: `bond_rewire`, `ring_adjust`, `methyl_shift`,
  `local_skeleton_rearrange`, `ch2_bridge_move`, `methylation`, `ring_expand`.
  Each product is
  validated as a connected single-bond hydrocarbon and canonicalized; products
  equal to the parent are rejected.
- Generated products are deduplicated against the known library plus the
  temporary in-task library, screened with the same rules, and routed with the
  same property pipeline.
- Output layout: one flat result table per stage in the run folder
  (`perturbed_candidates.csv`, `perturbation_screened.csv`,
  `perturbation_property_predictions.csv`,
  `tier1..tier4_candidates.csv`, `potential1..3_candidates.csv`) plus
  `perturbation_group_summary.csv`, which carries the per seed x operation
  counts. Each row keeps its `group_key`
  (`perturbed_from_hits/C<n>/seed_<i>/<operation>`) as provenance.
- Every campaign of a run folder shares those tables. A new campaign is merged
  into the stored products (same SMILES win, `candidate_id` numbering continues,
  the attempt log keeps every campaign), and the merged rows are re-scored with
  the current settings so one tier rule covers all of them. Untick "keep the
  products already in the output folder" (CLI: `--replace-existing-products`) to
  replace them instead.
- Merging refuses to mix two model versions: when the stored products were made
  with a different melting/boiling/flash fusion head (or a different e3nn/STA
  checkpoint) the run stops before generating anything. The panel preselects the
  fusion-head version recorded in the folder, so the default flow cannot hit it.
- The manual review pool merges the run's seeds with the Tier 1-3 perturbation
  products into `manual_review/candidates_for_manual_calculation.csv`.

### Iterative perturbation (targeted push towards Tier 1)

`--iterations N` (panel: "Iterative perturbation towards Tier 1") turns a
perturbation campaign into an optimisation loop. It is driven by a **static**
model, `train-perturbation-iteration-model` /
`Evaluation_Iteration/evaluation/model/perturbation_iteration/perturbation_delta_iteration.joblib`,
trained once from the data the run folder already holds: every parent/product
pair of the generation library and the first large campaign, i.e. the
property-change trend of the operator mix. Rounds never re-train it; it only
decides *where* and *how densely* the next round samples.

The v1 artifact records
`ring_count_definition=formula_cycle_rank_full_molecule_v1`. If an older
pre-release artifact lacks that contract marker, the iterative command retrains
it from the current run before scoring. This keeps the static model's ring
feature consistent with screening while avoiding a second feature-schema
version.

- Seed list: a work queue over Tier 2/3 and Potential 1/2/3. Every round reads
  those channel tables, removes parents that this run folder has already
  perturbed, and admits new products that land in one of the five pools. Tier 1
  never enters the queue because it has reached the target. Leads are ordered by
  JP-10 beat count and composite score before the per-round cap is applied.
- Sampling direction: the enumerated products of a round are scored by the
  model (predicted density / heating-value / melting-point margins of the child),
  and only the most promising `--per-round-budget` products - plus an
  exploration share - are screened and predicted.
- Sampling density: the attempts of a round are re-allocated between the six
  operators in proportion to the average gain each operator showed in the
  training data (never switching an operator off), so effort follows what
  actually moves the properties.
- Every round records `iteration_round`, `iteration_selected`,
  `iteration_predicted_beats` and `iteration_promise` on each product row, and
  the loop writes `iteration_summary.csv` (leads in the list, products screened,
  new Tier 1 per round, Tier 1 gain since the start) plus
  `iteration_pipeline_summary.json`.

## Novelty threshold calibration

`calibrate-novelty-threshold` reads a validation CSV containing
`fingerprint_novelty` plus either a precomputed error column or
actual/predicted property columns. By default, prediction error is relative
error, `abs(actual - predicted) / abs(actual)`. It scans candidate thresholds and
recommends the largest threshold whose below-threshold molecules still meet the
requested MAE and quantile-error limits.

```powershell
python -m fuel_design.cli calibrate-novelty-threshold --csv result\e3nn_validation.csv --actual-column target --predicted-column prediction --error-mode relative --max-mae 0.05 --max-quantile-error 0.10 --quantile 0.90
```

## Thesis-derived property helpers

- `estimate-thesis-density` reproduces the density-estimation path described in
  Section 5.2.3 of the thesis: RDKit 3D conformer, Bondi vdW radii with default
  scale `1.4`, exposed surface points, divergence-theorem vdW volume, then
  density in `g/cm3`.
- `heating-value-from-formation` applies the Section 5.2.2 heat-value equation
  after an external quantum-chemistry run supplies the fuel standard formation
  enthalpy (the external calculator itself is not part of the package).
- `train-sfs-vr-melting` / `predict-sfs-vr-melting` train and run the thesis
  Chapter 3 melting-point model from
  `Evaluation_Iteration/SFS-VR/SFS-VR-main/MP/MP-454-select-FEATURE - 15.xlsx` (15 precomputed Mordred features; Mordred itself is not required at runtime;
  voting weights XGBR 0.5, GBR 0.3, RFR 0.2; LR and SVR weighted to zero).
- `prepare-qm9-cbh-bdc-features` prepares QM9 hydrocarbon CBH-BDC fragment
  features (requires OpenBabel `obabel` in PATH);
  `prepare-cbh-bdc-formation-targets` converts QM9 `h298` into a
  formation-enthalpy training target through the atomization cycle;
  `train-cbh-bdc-delta-ml` trains Yang's CBH-BDC StackingRegressor and saves a
  `.joblib` model; `predict-cbh-bdc-formation` predicts formation enthalpy,
  heating value, and uncertainty from SMILES.
- Per-mirror-split folders are no longer materialised (they produced ~110 files
  per carbon in generation and ~19k files per perturbation campaign). The run
  keeps flat tables plus `group_summary.csv` (generation) and
  `perturbation_group_summary.csv` (perturbation), and
  `analysis/totals/` holds the merged channel pools
  (`tier1..tier4_candidates_total.csv` with the full molecule rows) together
  with `tier_summary.csv`, the whole-flow totals table and the final report.

```powershell
python -m fuel_design.cli estimate-thesis-density --smiles C1CC2C3CCC(C3)C2C1
python -m fuel_design.cli train-sfs-vr-melting
python -m fuel_design.cli predict-sfs-vr-melting --smiles CCCC --high-novelty
python -m fuel_design.cli heating-value-from-formation --smiles C --formation-enthalpy-kj-mol -74.87
python -m fuel_design.cli predict-cbh-bdc-formation --smiles C1CCCCC1
```

## Data layout

- `SCULPTOR_v1.0.0/` is the package root. `Generation/` holds the per-stage
  models and data, `Perturbation/` holds the perturbation line, and
  `Evaluation_Iteration/` holds the evaluation, rescoring, known-fuel matching
  and feedback loop. The shared code is in `core/` and `fuel_design/`.
- `dataset/` contains the GDB-13 table, the GDB-17 subset and QM9
  (`GDB/gdb13_g.csv`, `GDB/gdb17_500000.csv`, `QM9/...`).
- `Generation/screening/data/known_smiles_total_reference.txt` is the
  812,028-entry total reference library.
- `result/prod_full_2026_10_02/` contains the complete production result
  tables (candidate pools, stage ledger, iteration details and property
  predictions). `result/cinfogan_bench/`, `result/dft_validation/` and
  `result/cds_reference/` contain the external comparison, the DFT campaign
  and the CDS/NIST reference tables.
- ANI-1x raw coordinates are not redistributed; the shipped geometry scorer
  and the QM9 data are sufficient for the workflow included here.

## Package manifest and integrity

`manifest.json` lists every file in the full distribution. The SHA-256 values of
the three release assets are given both in the release notes and in `SHA256SUMS`.
After downloading the three archives into the package root, run

```powershell
powershell -ExecutionPolicy Bypass -File .\verify_sha256.ps1
```

to check them before extraction.


## Generation schedule

- `generation-schedule` returns curriculum parameters for a generation round.
- Rounds `1..20` use conservative warmup settings.
- Later rounds gradually increase `temperature`, `novelty_threshold`, and
  `base_diversity`.
- Optional `--novelty-rate` and `--valid-rate` apply feedback adjustment
  (validity guard / novelty boost).

## Optional dependencies

The project dependencies are installed in `.venv`.

Installed and verified:

- `rdkit`
- `torch`
- `torch-geometric`
- `pandas`
- `numpy`
- `pytest`
- `scikit-learn`, `xgboost`, `openpyxl`, `joblib`, `h5py`

## Notes for thesis reviewers

- The n1 linear-forest rule and the acyclic base-graph definition in this README
  match the current implementation; older drafts that mention "n1-n1 bonds only
  between adjacent indexes" or "base graphs may contain cycles" are obsolete.
- Same-source phase-fusion metrics (R2/MAE/RMSE on modelized labels) are
  reported as training diagnostics; independent experimental validation is kept
  separately in `experimental_data/organization_and_model_validation_20260805`.

