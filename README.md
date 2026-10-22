# Ranking Pareto-optimal designs: weighted-sum vs. ε-constraint vs. TOPSIS

A small, reproducible experiment on **multi-criteria decision-making (MCDM)**.
It generates a Pareto front for a synthetic satellite-configuration problem with
[pymoo](https://pymoo.org), then ranks the Pareto-optimal designs with three classic methods and
shows how the choice changes when the weights change.

> **The problem is synthetic.** The formulas are simplified engineering-style relations written for
> this demo. They are not real mission data, and the findings below describe this toy problem only.

## The question

Once an optimiser has found many Pareto-optimal designs, none is "best" by itself. How you pick one
depends on the method *and* on your preferences. How much does the pick depend on each?

## The problem

Five decision variables: solar-panel area, cell efficiency, battery capacity, battery energy density,
antenna diameter. Three objectives, all conflicting:

| Objective | Direction |
|---|---|
| Total mass [kg] | minimise |
| Total cost [kEUR] | minimise |
| Data volume [GB/day] | maximise |

One constraint: the battery must cover the eclipse load. See `src/problem.py`.

## Methods (`src/mcdm.py`)

- **Weighted-sum:** min-max normalise the criteria, then rank by the weighted sum.
- **TOPSIS:** rank by closeness to the ideal point and distance from the anti-ideal point
  (vector normalisation by default; a min-max variant is available via `norm="minmax"`).
- **ε-constraint:** minimise one criterion while capping the others. Feasible designs are ranked by the
  primary criterion; infeasible ones follow, ordered by how much they violate the caps.

## Run it

```bash
pip install -r requirements.txt
python src/run_experiment.py     # writes tables and figures to results/
python -m pytest -q              # 5 unit tests for the ranking methods
```

The run is seeded (`seed=1`), so the numbers below are reproducible.

## Results (120 Pareto-optimal designs)

![Pareto front and top picks](results/fig1_pareto_projections.png)

**Four weight scenarios (mass, cost, data):**

| Scenario | Weights | Weighted-sum top-1 | TOPSIS top-1 | Kendall τ (full rankings) | τ vs. TOPSIS with min-max |
|---|---|---|---|---|---|
| Balanced | 0.33 / 0.33 / 0.33 | #0 | #29 | -0.354 | 0.933 |
| Mass-focused | 0.6 / 0.2 / 0.2 | #2 | #0 | 0.930 | 0.928 |
| Cost-focused | 0.2 / 0.6 / 0.2 | #0 | #24 | 0.904 | 0.973 |
| Data-focused | 0.2 / 0.2 / 0.6 | #89 | #29 | 0.924 | 0.936 |

**Sweeping the cost weight from 0.02 to 0.96 (the rest split equally, 60 steps):**

![Weight sweep](results/fig2_weight_sweep.png)

- The weighted-sum top pick changed **2** times and TOPSIS changed **9** times.
- The two methods chose the same top design in **40%** of the sweep.
- Kendall τ between the two full rankings had a mean of 0.43 and ranged from -0.63 to 1.00.

**ε-constraint** (maximise data volume, mass capped at the 60th percentile, cost cap swept):

![Epsilon-constraint](results/fig3_epsilon_constraint.png)

The best achievable data volume rose from 2.2 to 282 GB/day as the cost cap grew from about 300 to 1,356 kEUR.
It rises in steps with plateaus, because only a few designs become feasible at each cap.

## What I take from it

1. **Normalisation mattered as much as the method.** Classic vector-normalised TOPSIS and the weighted sum
   disagreed strongly in the balanced scenario (τ = -0.35, no overlap in the top 10). TOPSIS with min-max
   normalisation agreed with the weighted sum (τ between 0.93 and 0.97 in all four scenarios). Much of the
   disagreement came from scaling, not from the aggregation rule.
2. **The weighted-sum pick jumps.** As the cost weight grew, it moved between the two ends of the front,
   while TOPSIS moved through intermediate designs. This fits the known limitation that a weighted sum
   struggles to reach points in non-convex parts of a front. I did not test convexity directly.
3. **Weights are a judgement, not a measurement.** A small shift in the weights can change the winner,
   so the weights and the normalisation should be reported next to any recommendation.
4. **ε-constraint is easy to explain to stakeholders** ("the cheapest design that stays under 800 kEUR"),
   but it needs sensible caps and does not produce a single ranking of its own.

## Limitations

- Synthetic problem, one random seed, three objectives. Real trade-off studies have more criteria, uncertainty and
  expert-set weights.
- The front comes from NSGA-II (a finite approximation); near-duplicates were removed.
- I compared methods with agreement measures (Kendall τ, top-k overlap), not with ground truth. There is no "correct" ranking.

## Next steps

- Add more methods (AHP, PROMETHEE, VIKOR) and a rank-reversal test.
- Add uncertainty: sample the weights and report how often each design wins.
- Apply the same code to a real set of configurations from a trade-off study.

## Files

```
src/problem.py          synthetic satellite problem (pymoo)
src/mcdm.py             weighted-sum, TOPSIS, ε-constraint
src/run_experiment.py   builds the front, ranks it, writes results/
tests/test_mcdm.py      unit tests
results/                CSV tables, figures, console output
```
