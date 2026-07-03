# steer-opencell-topdown

A top-down optimization layer on top of [`steer-opencell-design`](https://github.com/stanford-developers/steer-opencell-design).

Instead of going **design → performance/cost** (OpenCell's native direction), this app inverts the problem: given target performance and cost constraints, find the optimal cell design parameters.

---

## Where This Fits (STEER Ecosystem)

```
[steer-opencell-synthesis]      synthesis conditions → material properties
        ↓
[steer-opencell-design]         cell design params → performance & cost   ← wraps this
        ↓
[steer-opencell-topdown]  ✦     performance/cost targets → optimal design  ← this repo
        ↓
[steer-opencell-ESS]            optimized cell design → pack/system-level cost & performance
```

---

## Problem / Motivation

- OpenCell computes performance and cost from design parameters (forward direction)
- In practice, researchers and engineers start from a target: "I need ≥ 250 Wh/kg at ≤ $80/kWh — what design gets me there?"
- Manually sweeping the design space by hand is slow and misses non-obvious optima
- This app wraps OpenCell as a black-box simulator and searches the design space systematically

---

## Goals

- Accept user-specified target metrics (e.g. energy density, cost/kWh, cycle life) as constraints or objectives
- Return candidate cell designs (parameter sets) that satisfy or approximate those targets
- Surface the **Pareto front** between competing objectives (energy density vs. cost) rather than a single point
- Be modular: optimization method is swappable (grid search → Monte Carlo → Bayesian optimization)

## Non-Goals (for MVP)

- Not modeling pack/system-level (ESS) performance — that's `steer-opencell-ESS`
- Not optimizing synthesis conditions — that's `steer-opencell-synthesis`
- Doesn't need to be real-time or interactive — batch optimization is fine to start

---

## Proposed Approach

Treat OpenCell as a **black-box forward simulator**. Wrap it in a search loop:

```
propose design params
        ↓
evaluate via OpenCell → (energy_density, cost, cycle_life, ...)
        ↓
check against targets / update optimizer
        ↓
repeat until converged or budget exhausted
```

### Optimization methods (planned, in order of complexity)

| Method | Status | Notes |
|---|---|---|
| Grid search | MVP | Simple baseline; good for 1-2 parameter sweeps + Pareto plot |
| Monte Carlo sampling | v0.2 | Scales better to higher dimensions |
| Bayesian optimization | v0.3 | Most sample-efficient; ref: Attia et al. 2020 (Nature) |

---

## Key Design Parameters (Search Space)

Candidates for the ~8–10 settable parameters drawn from OpenCell's inputs:

**Electrode**
- `mass_loading` (mg/cm²) — active material coating weight
- `calender_density` (g/cm³) — electrode compression
- `coating_thickness` (µm) — derived from above; thicker = more capacity, longer diffusion path
- `N/P ratio` — anode/cathode capacity ratio (typically 1.05–1.20 for graphite)

**Chemistry**
- Cathode: LFP / NMC622 / NMC811 / NCA / Na-ion (NFM)
- Anode: synthetic graphite / hard carbon / Li metal

**Geometry**
- Cell format: cylindrical / prismatic / pouch
- Cell dimensions, separator thickness/porosity, electrolyte type, tab design

---

## MVP Scope

| | Description |
|---|---|
| **Input** | Target range for 1–2 key metrics (energy density, cost/kWh); optional fixed chemistry to reduce search space |
| **Output** | Candidate designs + computed performance/cost; Pareto front plot |
| **Out of scope (v0)** | Multi-chemistry sweep, manufacturing constraints, real supply-chain pricing |

---

## Validation

- Optimizer outputs must be physically plausible — check against `cell_references/` realistic parameter ranges from `steer-opencell-design`
- Sanity check: results should reproduce known LFP vs. NMC811 trade-off curve

---

## Open Questions (to resolve with Nick)

- [ ] Which 1–2 metrics for the first demo? (energy density vs. cost is the obvious pair)
- [ ] Is cathode chemistry a fixed user choice, or part of the search space itself?
- [ ] Go straight to Monte Carlo baseline, or start with a simpler grid search first?

---

## First Milestone

- [ ] Run `steer-opencell-design` quickstart end-to-end locally
- [ ] Identify the ~8–10 most impactful settable parameters + realistic ranges (from `cell_references/`)
- [ ] Implement grid search over 2 parameters → Pareto front plot for energy density vs. cost
- [ ] That's a demoable v0 to bring to Nick

---

## Setup

```bash
# Clone and install the core dependency
pip install steer-opencell-design

# Clone this repo
git clone https://github.com/PTLin84/steer-opencell-topdown
cd steer-opencell-topdown

# Install dependencies (once requirements.txt exists)
pip install -r requirements.txt
```

---

## References

- Attia et al. (2020). *Closed-loop optimization of fast-charging protocols for batteries with machine learning.* Nature, 578, 397–402.
- Severson et al. (2019). *Data-driven prediction of battery cycle life before capacity degradation.* Nature Energy, 4, 383–391.
- Yao, Benson & Chueh (2025). *Critically assessing sodium-ion technology roadmaps and scenarios for techno-economic competitiveness against lithium-ion batteries.* Nature Energy, 10, 404–416.
- Pegel, Jany & Sauer (2025). *Pareto-Optimal Design of Automotive Battery Systems with Tabless Cylindrical Lithium-Ion Cells.* Energy Technology, 13, 2401479.
