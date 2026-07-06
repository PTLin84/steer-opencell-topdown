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

*These are assumptions that need to be verified.*

- OpenCell computes performance and cost from design parameters (forward direction)
- In practice, researchers and engineers start from a target: "I need ≥ 250 Wh/kg at ≤ $80/kWh — what designs can get me there with the least costs?"
- Manually sweeping the design space by hand is slow and misses non-obvious optima
- This app wraps OpenCell as a black-box simulator and searches the design space systematically


---

## Goals (for MVP)

- Accept user-specified target metrics (e.g. energy density, cost/kWh) as constraints or objectives
- Return candidate cell designs (parameter sets) that satisfy or approximate those targets
- Surface the **Pareto front** between competing objectives (energy density vs. cost) rather than a single point
- Be modular: optimization method is swappable (grid search → Monte Carlo → Bayesian optimization)

## Goals (Future iterations)

- Make the design process agentic and self-correcting.
- Take into account the manufacturing costs, not just BOM costs.

---

## Proposed Approach

Treat OpenCell as a **black-box forward simulator**. Wrap it in a search loop:

```
propose design params
        ↓
evaluate via OpenCell → (energy_density, cost_per_kWh, ...)
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

For MVP v1: **enumerate chemistry as the outer loop** — run one 3D grid per chemistry and overlay the resulting Pareto fronts. All other discrete parameters (anode material, cell format) are fixed at defaults from `cell_references/`.

---

## Parameter space strategy

The full design space has 10+ dimensions. Parameters are tiered by their leverage on `specific_energy` and `cost_per_energy`:

| Tier | Parameters | Treatment |
|---|---|---|
| **Sweep** | `mass_loading`, `calender_density`, N/P ratio | 3D grid (continuous) |
| **Enumerate** | Cathode chemistry (LFP / NMC622 / NMC811) | Outer loop — one Pareto plot per chemistry |
| **Fix at defaults** | Separator thickness/porosity, binder/CA ratios, electrolyte overfill, CC geometry | Values from `cell_references/` |
| **Fix at one format** | Cell format, container dimensions | Cylindrical, frozen |

> N/P ratio is not swept independently — it is derived from cathode and anode specific capacities:
> `anode_mass_loading = (cathode_mass_loading × cathode_specific_capacity) / (N/P × anode_specific_capacity)`

---

## MVP v1 Scope

**Goal:** validate that the Pareto front idea makes sense at all — does a meaningful trade-off surface emerge from this search space?

| | Description |
|---|---|
| **Search space** | 3D grid: `mass_loading` × `calender_density` × N/P ratio (e.g. 10×10×5 = 500 evaluations) |
| **Chemistry loop** | LFP, NMC622, NMC811 — one Pareto front per chemistry, overlaid on the same plot |
| **Output** | `(specific_energy, cost_per_energy)` for each grid point; Pareto front per chemistry |
| **Fixed** | Anode: Synthetic Graphite; format: Cylindrical; all other params at `cell_references/` defaults |
| **Out of scope** | Manufacturing constraints, real supply-chain pricing, cycle life |

---

## Validation

- All grid points must fall within physically plausible parameter ranges — cross-check against `cell_references/` in `steer-opencell-design`
- Sanity check: LFP Pareto front should sit lower-left (lower energy density, lower cost) and NMC811 upper-right vs. NMC622. If the fronts are jumbled, something is wrong with the parameter ranges or the N/P derivation.

---

## Open Questions (to discuss with Nick)

- Do we now how people in the industry design a new cell?
    - Do they tend to start with a commercial cell then tweak?
    - What are the costs of changing cell dimensions?
    - Do people design and outsource manufacturing?
    - Any contacts to ask these questions?
- Does it make sense to treat OpenCellDesign a black box? 
        - Or should we build formulas instead for this top-down approach?
- What are the parameters that matter the most when searching?
        - I currently choose `Cathode chemistry`, `cathode_mass_loading`, `cathode_calender_density`, and `N/P ratio`.
- Can we provide cycle life predictions?

---

## Milestones

### MVP v1 — validate the Pareto front concept

- [ ] Implement the N/P ratio derivation: compute `anode_mass_loading` from cathode loading + target N/P ratio + chemistry-specific capacities
- [ ] Implement a 3D grid search over `mass_loading` × `calender_density` × N/P ratio (e.g. 10×10×5 per chemistry)
- [ ] Enumerate across LFP, NMC622, NMC811 as the outer loop
- [ ] Collect `(specific_energy, cost_per_energy)` for each grid point; skip physically invalid points (e.g. negative porosity)
- [ ] Extract and overlay Pareto fronts per chemistry on a single plot
- [ ] **Done when**: the three chemistry fronts are clearly separated and qualitatively match expectations (LFP cheapest, NMC811 most energy-dense)

### v1.1 — improve search efficiency

- [ ] Switch from grid search to Monte Carlo sampling for more efficient coverage of the 3D space
- [ ] Add a sensitivity screen: perturb each parameter ±20% from a baseline cell and rank influence on `specific_energy` and `cost_per_energy` — use this to validate the tier assignments above

### v1.2 — Bayesian optimization

- [ ] Integrate a BO library (e.g. `botorch`, `scikit-optimize`, or `ax-platform`)
- [ ] Benchmark sample efficiency vs. grid search / Monte Carlo on the same problem

> **Note on cycle life:** OpenCell has no degradation model — `capacity_loss` in the library is a static geometric quantity (capacity at the voltage cutoff), not cycle-to-cycle fade. Adding cycle life as an optimization objective would require an external empirical model (e.g., Severson et al. 2019) that takes OpenCell outputs as inputs. Scope TBD.

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
