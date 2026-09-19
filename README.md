# Capacitated Vehicle Routing Problem — Exact and Heuristic Methods

Optimization Methods programming assignment. We model last-mile delivery
routing for a Colombo distributor as a Capacitated Vehicle Routing Problem
(CVRP) and solve it three ways: an exact Mixed-Integer Linear Program, a
constructive heuristic, and a metaheuristic. The three are compared on
solution quality, runtime, scalability and feasibility.

| | |
|---|---|
| **Members** | Nisal — data pipeline, exact ILP, experiment infrastructure<br>Ayesh — Clarke–Wright savings, genetic algorithm, comparative analysis |
| **Problem class** | CVRP (NP-hard; TSP reduces to it) |
| **Exact method** | MTZ Mixed-Integer Linear Program, solved with CBC via PuLP |
| **Heuristics** | Clarke–Wright savings; Genetic Algorithm with capacity-respecting split |
| **Data** | CVRPLIB Augerat benchmark sets; a 30-stop Colombo instance with OSM road distances; synthetic instances for the scalability study |

---

## Problem

A single depot serves *n* customers, each with a known demand *qᵢ*. A
homogeneous fleet of *K* vehicles, each of capacity *Q*, must deliver to
every customer exactly once. Each vehicle starts and ends at the depot and
may not exceed its capacity. Minimise total distance travelled.

The full mathematical model — sets, parameters, decision variables,
objective, and constraints C1–C6 including the MTZ subtour elimination — is
in [`report/formulation.md`](report/formulation.md). Every constraint in
`src/exact_ilp.py` is labelled with the tag it carries in that document.

---

## Repository layout


---

## Setup

Requires Python 3.10 or newer.

```bash
git clone <repo-url>
cd cvrp-optimization
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

CBC ships with PuLP, so no separate solver installation is needed.

Download the benchmark instances into `data/instances/`:

```bash
python src/download_instances.py       # or place .vrp files there manually
```

---

## Usage

Solve one instance with the exact method:

```python
import sys; sys.path.insert(0, "src")
from data_loader import load_cvrplib
from model import describe
import exact_ilp

inst = load_cvrplib("data/instances/A-n32-k5.vrp")
sol = exact_ilp.solve(inst, time_limit=600)
print(describe(sol))
```

Build the real-world Colombo instance (run once, then commit the JSON):

```bash
python src/build_osm_instance.py --out data/instances/colombo-30.json
```

Run the experiments:

```bash
python src/run_experiments.py --quick       # smoke test, a few seconds
python src/run_experiments.py --benchmark   # CVRPLIB validation
python src/run_experiments.py --full        # the run reported in the paper
```

Generate the figures:

```bash
python src/analyze.py --results results/results.csv --out results/figures
```

---

## The solver interface

Every solver exposes the same signature. This is what lets the experiment
runner treat all three methods identically, and it is the reason the three
objective values are directly comparable.

```python
def solve(instance: Instance, time_limit: float = 300.0, **kwargs) -> Solution
```

`Solution.objective` is always computed by `model.solution_cost`. No solver
computes its own cost — a duplicated cost function is how two methods end up
being compared on two different scales without anyone noticing.

`Solution.status` distinguishes `optimal` (proved), `feasible`, `timeout`
and `infeasible`. Only a proved optimum is used as the reference when
computing heuristic gaps.

---

## Results

Full tables and figures in `results/`. Headline findings:

- The exact MILP proves optimality on small instances and matches published
  CVRPLIB optima, but the MTZ formulation's weak LP relaxation makes it
  intractable beyond roughly *n* = 20 within a 15-minute limit.
- The metaheuristic returns solutions within a small percentage of optimal
  in a fraction of the time, and keeps returning feasible solutions at sizes
  where the exact method returns nothing at all.
- Reported gaps are measured against proved optima where available and
  against published best-known values otherwise; the `reference_type` column
  in `results.csv` records which was used for every row.

---

## Reproducibility

- Every stochastic result is averaged over 5 fixed seeds and reported as
  mean ± standard deviation.
- `data/instances/colombo-30.json` is committed rather than rebuilt at run
  time, so no experiment depends on a live network call.
- CVRPLIB distances are rounded to the nearest integer, matching the EUC_2D
  convention the published optima are computed under.

---

## Validation

The exact solver was verified two ways:

1. Against exhaustive enumeration on instances small enough to brute-force
   (*n* ≤ 7), for both the exactly-*K* and at-most-*K* fleet variants.
2. Against published CVRPLIB optima on the Augerat instances.

Both checks are in `notebooks/02_exact.ipynb`.

---

## References

Full reference list in the report. Primary sources: Miller, Tucker &
Zemlin (1960) for the subtour formulation; Clarke & Wright (1964) for the
savings heuristic; Augerat et al. (1995) for the benchmark instances;
Toth & Vigo (2014), *Vehicle Routing: Problems, Methods, and Applications*.
