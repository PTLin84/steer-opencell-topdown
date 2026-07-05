# steer-opencell-topdown

A top-down optimization layer on top of [`steer-opencell-design`](https://github.com/stanford-developers/steer-opencell-design).

Instead of going **design → performance/cost** (OpenCell's native direction), this app inverts the problem: given target performance and cost constraints, find the optimal cell design parameters.

---

## Where This Fits (STEER Ecosystem)

```
[steer-opencell-synthesis]  →  [steer-opencell-design]  →  [steer-opencell-ESS]
  synthesis → materials          design → performance            cell → system
                                         ↑
                              [steer-opencell-topdown]  ✦
                              inverts the core layer:
                              targets → optimal design
```

`steer-opencell-topdown` is a **wrapper around** `steer-opencell-design`, not a downstream layer. It calls OpenCell repeatedly as a black-box simulator while searching the design space.

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
evaluate via OpenCell → (energy_density, cost_per_kWh, cycle_life, ...)
        ↓
check against targets / update optimizer
        ↓
repeat until converged or budget exhausted
```

### Optimization methods (planned, in order of complexity)

| Method | Status | Notes |
|---|---|---|
| Grid search | MVP | Simple baseline; 2-parameter sweep + Pareto plot |
| Monte Carlo sampling | v0.2 | Scales better to higher dimensions |
| Bayesian optimization | v0.3 | Most sample-efficient; ref: Attia et al. 2020 (Nature) |

---

## Key Design Parameters (Search Space)

The independent inputs to OpenCell — these are the parameters the optimizer actually varies.

**Continuous (sweep these)**

| Parameter | Unit | Typical range | Notes |
|---|---|---|---|
| `mass_loading` | mg/cm² | 10–30 | Primary lever for energy density |
| `calender_density` | g/cm³ | 2.5–4.0 | Higher = better energy density, worse rate capability |
| `N/P ratio` | — | 1.05–1.20 | Must stay > 1.0 to prevent Li plating |
| Separator thickness | µm | 12–25 | Affects safety and ion conductivity |

> Note: `coating_thickness` is **derived** from `mass_loading` ÷ `calender_density` — it is not an independent input and should not be varied directly.

**Discrete (fix or enumerate)**

| Parameter | Options | Notes |
|---|---|---|
| Cathode chemistry | LFP / NMC622 / NMC811 / NCA | Biggest single driver of energy density and cost |
| Anode material | Graphite / Hard carbon / Li metal | Li metal = solid-state only |
| Cell format | Cylindrical / Prismatic / Pouch | Affects geometry constraints |

For MVP: **fix cathode chemistry as a user choice** to keep the continuous search space to 2–3 parameters. Enumerate across chemistries in v0.2.

---

## MVP Scope

| | Description |
|---|---|
| **Input** | Cathode chemistry (user picks one) + target range for energy density (Wh/kg) and cost ($/kWh) |
| **Output** | Grid of candidate designs with computed energy density and cost; Pareto front plot |
| **Out of scope (v0)** | Multi-chemistry sweep, cycle life optimization, manufacturing constraints, real supply-chain pricing |

---

## Validation

- All optimizer outputs must fall within physically plausible parameter ranges — cross-check against `cell_references/` in `steer-opencell-design`
- Sanity check: with chemistry fixed to LFP vs. NMC811, the resulting Pareto fronts should qualitatively match the known cost/energy trade-off (LFP: lower energy density, lower cost; NMC811: higher energy density, higher cost)

---

## Open Questions (to resolve with Nick)

- [ ] Which metrics for the first demo? (energy density vs. cost is the obvious pair; cycle life as a third axis in v0.2?)
- [ ] Is cathode chemistry a fixed user choice for MVP, or should it be part of the sweep from day one?
- [ ] Start with grid search baseline, or go straight to Monte Carlo?

---

## Milestones

### Before Nick sync — unblock yourself

- [ ] `pip install steer-opencell-design` and run the quickstart end-to-end
- [ ] Modify `mass_loading` and `N/P ratio`, call `propagate_changes()`, confirm `cell.energy` and `cell.cost_per_energy` update correctly
- [ ] Browse `cell_references/` to nail down realistic parameter ranges for the grid
- [ ] Decide: fix cathode to NMC811 for the first demo, or run LFP + NMC811 side by side?

### MVP (v0) — demoable grid search

- [ ] Implement a 2D grid search over `mass_loading` × `N/P ratio` (e.g. 10×10 grid, fixed NMC811 cathode)
- [ ] Collect `(energy_density_Wh_kg, cost_per_kWh)` for each grid point
- [ ] Extract and plot the Pareto front (energy density vs. cost)
- [ ] Validate: does the Pareto front shape match qualitative expectations for NMC811?
- [ ] **Done when**: you can hand Nick a plot that clearly shows the trade-off curve and 2–3 specific designs on the front with their full parameter sets

### v0.2 — extend search space

- [ ] Add `calender_density` as a third search dimension
- [ ] Enumerate across cathode chemistries (LFP, NMC622, NMC811) and overlay their Pareto fronts
- [ ] Switch from grid search to Monte Carlo sampling (scipy or numpy) for more efficient coverage

### v0.3 — Bayesian optimization

- [ ] Integrate a BO library (e.g. `botorch`, `scikit-optimize`, or `ax-platform`)
- [ ] Benchmark sample efficiency vs. grid search / Monte Carlo on the same problem
- [ ] Add cycle life as a third objective (3D Pareto surface)

---

## Setup

```bash
# Install the core dependency
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
