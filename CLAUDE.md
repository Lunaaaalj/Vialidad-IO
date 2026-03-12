# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

This project uses **two Python environments**:

- `.venv/` (Python 3.14) — general data science stack (matplotlib, numpy, pandas, Pillow, Jupyter)
- `/Users/angelluna/Documents/STEM_PROJECTS/py/analisis-vial/venv/` — contains **Mesa 3.5** required for multiagent notebooks

To run notebooks that use Mesa (`modelL.ipynb`, `CSP_MultiAgent.ipynb`), the kernel must point to the `analisis-vial` venv. The `FutureWarning` about `seed` vs `rng` in Mesa output is expected and non-breaking.

### Launching Jupyter

```bash
# From the analisis-vial venv (needed for Mesa notebooks)
/Users/angelluna/Documents/STEM_PROJECTS/py/analisis-vial/venv/bin/jupyter notebook

# From the project venv (CSP-only notebook)
source .venv/bin/activate && jupyter notebook
```

### Compiling the formal model

```bash
cd models && latexmk -pdf main.tex
```

## Architecture

The project models a signalised traffic intersection using two complementary approaches that are integrated in `CSP_MultiAgent.ipynb`.

### CSP Layer (`CSP_Traffic.ipynb`)

- **Variables**: `W`, `E`, `NW`, `NE` — binary (0=Red, 1=Green) state of each arm
- **Constraints**: No two crossing arms can be green simultaneously (avenue arms conflict with diagonal arms); at least one arm must be green (liveness)
- **Solver**: hand-written recursive backtracking (`backtrack()` + `is_valid()`)
- **Output**: list of `dict` solutions → converted to `list[frozenset[Arm]]` for use by the controller

The formal mathematical definition lives in `models/main.tex` (compiled to `models/main.pdf`).

### Multiagent Layer (`modelL.ipynb`)

Built on **Mesa 3.5**. Three classes:

| Class | Role |
|---|---|
| `Arm` (Enum) | Maps arm names (`W`, `E`, `NW`, `NE`) to intersection geometry |
| `TrafficLight` | One per arm; holds `state = "green" \| "red"`; phase control is external |
| `Vehicle` | Follows a 5-waypoint route (entry → stop-line → center → stop-line → exit); stops at waypoint 1 when light is red |
| `Cruce` (Model) | Spawns vehicles on a schedule; runs adaptive phase control each step |

**Geometry**: intersection center at `(10, 10)`, plaza circle radius `R=2.2`. `ENTRY_POINT`, `CIRCLE_POINT`, and `ROUTES` dicts are derived from arm angles and drive all movement.

**Phase control in `Cruce.step()`**: switches to the `_best_phase()` (CSP-valid phase with longest queue) after `min_green` steps, or forces switch after `max_green` steps.

### Integration (`CSP_MultiAgent.ipynb`)

`Cruce` accepts `csp_phases: list[frozenset[Arm]]` — the exact output of the CSP solver — as its set of allowable phases. This is the only coupling point between the two layers. Passing `FIXED_PHASES` (only avenue + diagonal) reproduces the original 2-phase behaviour for comparison.

### Outputs

- `reports/csp_configurations*.png` — grid of valid CSP configurations
- `reports/figures/intersection_simulation.gif` — animated simulation (requires Pillow)
- `reports/figures/csp_multiagent_simulation.gif` — combined model animation
- `reports/csp_multiagent_metrics.png` / `reports/csp_vs_fixed_comparison.png` — metric plots
