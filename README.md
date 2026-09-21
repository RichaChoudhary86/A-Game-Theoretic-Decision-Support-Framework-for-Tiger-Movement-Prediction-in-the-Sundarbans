# Tiger movement prediction in the Sundarbans: game-theoretic framework and baseline-controlled evaluation

Code accompanying the manuscript *"A Game-Theoretic Decision-Support Framework for Tiger Movement
Prediction in the Sundarbans: Evaluation Against Naive Baselines and the Role of Temporal Resolution"*.

<!-- TODO(authors): add article DOI / journal reference and the Zenodo DOI badge -->

The pipeline couples (1) a CART next-cell model on a 2 km grid, (2) a Stackelberg-style leader payoff
obtained by Monte-Carlo perturbation of the fix, and (3) a three-state finite automaton
(Resting / Moving / Hunting), and evaluates it against a majority-class baseline using temporally
blocked cross-validation and leave-one-individual-out testing.

## Repository layout

```
src/tigermove/     library code
  data.py            telemetry loading / cleaning, prey centroids
  grid.py            2 km grid, cell indexing, direction labels (Center/N/S/E/W)
  features.py        day/year sin-cos, prey-proximity distances, NDVI / land-use
  payoff.py          CART model, Monte-Carlo payoff, argmax decision rule, trajectory roll-out
  automaton.py       finite automaton (Table VI)
  evaluate.py        resolution diagnostic, blocked CV, leave-one-individual-out, ablation
scripts/           command-line entry points (below)
tests/             unit tests (synthetic data; no telemetry required)
results/           reference outputs produced by the scripts
data/              data description + prey centroids (raw telemetry is NOT included)
legacy/            original exploratory notebooks, kept for provenance (see legacy/README.md)
```

## Installation

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m pytest -q tests                              # synthetic-data tests
```

## Data

Raw GPS telemetry and prey occurrence records belong to the Wildlife Institute of India and are not
redistributed here. See [`data/README.md`](data/README.md) for access and the expected file layout.

## Reproducing the results

```bash
# Tables VII, VIII, IX (native vs daily resolution; blocked CV and leave-one-out vs baseline; ablation)
python scripts/run_evaluation.py --data-dir data/raw --out results

# Robustness of the headline comparison to where the 2 km grid is anchored
python scripts/run_sensitivity.py --data-dir data/raw --n-offsets 30

# Displacement diagnostics behind the 2 km cell size
python scripts/run_displacement.py --data-dir data/raw

# Regression benchmark (regenerated Table IV), from the 'all tigers, all prey' feature table
python scripts/run_benchmark.py --input path/to/All_tiger_all_prey.csv
```

`run_evaluation.py` uses 10,000 Monte-Carlo draws per transition and takes a few minutes on a laptop;
`--n-draws` and `--seed` control the Monte-Carlo step. All randomness is seeded.

## Method summary and design decisions

* **Data.** Only covariate-complete fixes (matched NDVI and land-use) are used. Fixes outside the
  study-area bounding box are removed (one GPS error at 93.48 E is removed from collar 7831).
* **Grid.** Regular 2 km equirectangular lattice anchored at the south-west corner of the pooled
  extent (`Grid.from_fixes`). Results depend on the grid registration; `run_sensitivity.py` quantifies this.
* **Daily resolution.** The first fix of each UTC day is kept; transitions are formed **only between
  consecutive calendar days**.
* **Direction labels.** Compass sector of the vector between consecutive cell centres
  (N [315,45), E [45,135), S [135,225), W [225,315) degrees); no change of cell is `Center`.
* **Payoff.** For each transition, the position is perturbed 10,000 times by an independent
  uniform(-0.0007, 0.0007) degree offset in latitude and longitude (each draw perturbs the original
  fix); the payoff of an action is its empirical probability under the fitted CART
  (`DecisionTreeClassifier`, unpruned) and the leader takes the argmax. Prey-proximity covariates are
  recomputed from the perturbed position. This is a reduced-form leader rule, **not** a solved
  two-agent Stackelberg equilibrium.
* **Validation.** `TimeSeriesSplit(5)` (expanding window, contiguous later block as test) within
  collar and on the chronologically pooled transitions; leave-one-individual-out trains on the other
  collars and predicts the held-out collar in full. The baseline predicts the modal training-set outcome.
  Reported: accuracy, macro-F1 over the five outcome classes, Wilson 95% CI, exact one-sided binomial
  test of accuracy against baseline accuracy.
* **Automaton.** Transition table exactly as in Table VI. The manuscript does not define the `H` input
  operationally; `symbols_from_trajectory` raises it when the predicted cell centre lies within
  `hunt_radius_km` of a prey centroid (assumption, adjustable).

## Known limitations

* Two well-sampled collars (128 and 77 daily transitions) plus one short series (15); confidence
  intervals are wide.
* Prey covariates are distances to fixed annual occurrence centroids, hence a deterministic function of
  position (no prey dynamics). NDVI / land-use are evaluated at the realised fix, not at candidate cells.
  

## License and citation

MIT License (see `LICENSE`). Please cite the article (see `CITATION.cff`).
