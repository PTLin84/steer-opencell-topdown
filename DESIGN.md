# steer-opencell-topdown — Code Design

This document describes the intended code architecture for the top-down optimizer. The system is designed as a **backend API** from the start, with a Plotly Dash frontend to be layered on later.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  Dash Frontend (future)         │
│          (controls, tables, Plotly charts)      │
└─────────────────────┬───────────────────────────┘
                      │ HTTP / function call
┌─────────────────────▼───────────────────────────┐
│                  Backend API                    │
│   /search   /results   /figures   /status       │
└──────┬──────────────┬──────────────┬────────────┘
       │              │              │
┌──────▼──────┐ ┌─────▼──────┐ ┌─────▼────────────┐
│  Optimizers │ │  Results   │ │  Figure Builders │
│  (grid /    │ │  (schema,  │ │  (Plotly traces, │
│  MC / Bayes)│ │  Pareto)   │ │   layouts)       │
└──────┬──────┘ └────────────┘ └──────────────────┘
       │
┌──────▼──────────────────────────────────────────┐
│              CellSimulator                      │
│  (wraps steer-opencell-design as black box)     │
└─────────────────────────────────────────────────┘
```

All layers are independently testable. The optimizer doesn't know about figures; the figure builder doesn't know about optimizers — it only sees a results dataframe.

---

## Module Structure

```
steer_opencell_topdown/
├── __init__.py
├── core/
│   ├── simulator.py        # OpenCell wrapper: params → CellResult
│   ├── search_space.py     # Parameter bounds, N/P derivation, validation
│   └── pareto.py           # Pareto front extraction (pure function)
├── optimizers/
│   ├── base.py             # Abstract BaseOptimizer interface
│   ├── grid_search.py      # GridSearchOptimizer
│   ├── monte_carlo.py      # MonteCarloOptimizer
│   └── bayesian.py         # BayesianOptimizer (v1.2)
├── results/
│   ├── schema.py           # Dataclasses: SearchConfig, CellResult, RunResult
│   └── figures.py          # Plotly figure builders
└── api/
    └── routes.py           # HTTP endpoints (Flask / FastAPI)
```

---

## Core Abstractions

### 1. `SearchConfig` — defines what to run

```python
@dataclass
class ParameterBounds:
    mass_loading: tuple[float, float]       # mg/cm², e.g. (10.0, 30.0)
    calender_density: tuple[float, float]   # g/cm³, e.g. (2.5, 4.0)
    np_ratio: tuple[float, float]           # unitless, e.g. (1.05, 1.20)

@dataclass
class SearchConfig:
    # Outer loop — one Pareto run per chemistry
    chemistries: list[str]                  # e.g. ["LFP", "NMC622", "NMC811"]
    anode_chemistry: str                    # fixed, e.g. "Synthetic Graphite"
    cell_format: str                        # fixed, e.g. "cylindrical"

    # Continuous search space
    bounds: ParameterBounds

    # Optimizer selection
    optimizer: Literal["grid", "monte_carlo", "bayesian"]

    # Optimizer hyperparams
    grid_resolution: tuple[int, int, int]   # e.g. (10, 10, 5) for grid search
    n_samples: int                          # for Monte Carlo
    n_iterations: int                       # for Bayesian

    # Objectives (must be valid CellResult field names)
    objectives: list[str] = field(
        default_factory=lambda: ["specific_energy", "cost_per_energy"]
    )
```

### 2. `CellResult` — output of a single forward evaluation

```python
@dataclass
class CellResult:
    # Input params (recorded for traceability)
    chemistry: str
    mass_loading: float         # mg/cm²
    calender_density: float     # g/cm³
    np_ratio: float

    # Derived (not independently set)
    anode_mass_loading: float   # mg/cm², derived from N/P ratio
    coating_thickness: float    # µm, derived

    # Outputs
    specific_energy: float      # Wh/kg
    volumetric_energy: float    # Wh/L
    cost_per_energy: float      # $/kWh
    cost: float                 # $ (total BOM)
    mass: float                 # g
    energy: float               # kWh
    reversible_capacity: float  # Ah

    # Status
    valid: bool                 # False if OpenCell raised (e.g. negative porosity)
    error: str | None           # error message if not valid
```

### 3. `RunResult` — output of a full search run

```python
@dataclass
class RunResult:
    config: SearchConfig
    candidates: pd.DataFrame    # one row per CellResult (valid + invalid)
    pareto: pd.DataFrame        # Pareto-optimal subset (valid only)
    run_id: str                 # UUID for API tracking
    elapsed_seconds: float
    n_evaluated: int
    n_valid: int
```

---

## CellSimulator

Single responsibility: translate a flat `dict` of design parameters into a `CellResult` by constructing and evaluating an OpenCell cell object.

```python
class CellSimulator:
    """Wraps steer-opencell-design as a stateless black-box forward pass."""

    def __init__(self, cell_format: str = "cylindrical"):
        self._cell_format = cell_format
        self._reference_cells = {}   # cache: chemistry → baseline cell from cell_references/

    def evaluate(
        self,
        chemistry: str,
        mass_loading: float,
        calender_density: float,
        np_ratio: float,
    ) -> CellResult:
        """
        Build a cell with the given parameters and return computed metrics.
        Returns CellResult with valid=False on OpenCell errors (e.g. negative porosity).
        """
        ...

    def _derive_anode_mass_loading(
        self,
        cathode_mass_loading: float,
        np_ratio: float,
        cathode_chemistry: str,
        anode_chemistry: str,
    ) -> float:
        """
        anode_ml = (cathode_ml × cathode_specific_capacity)
                   / (np_ratio × anode_specific_capacity)
        """
        ...

    def _get_reference_cell(self, chemistry: str) -> ocd.CylindricalCell:
        """
        Load baseline cell for chemistry from cell_references/.
        Used to inherit fixed params (CC geometry, separator, electrolyte).
        """
        ...
```

**Design note:** The simulator loads a reference cell for each chemistry from `cell_references/` and mutates only the swept parameters. This ensures all the fixed parameters (CC geometry, separator, electrolyte, encapsulation) stay physically consistent with real measured cells, rather than being invented.

---

## Optimizers

All optimizers share the same interface so they are interchangeable.

```python
class BaseOptimizer(ABC):

    def __init__(self, simulator: CellSimulator):
        self.simulator = simulator

    @abstractmethod
    def run(self, config: SearchConfig) -> RunResult:
        """Execute the search and return results."""
        ...

    def _evaluate_all(
        self, params_list: list[dict], config: SearchConfig
    ) -> list[CellResult]:
        """Evaluate a list of parameter dicts. Catches errors per point."""
        ...
```

### GridSearchOptimizer

```python
class GridSearchOptimizer(BaseOptimizer):
    """
    Enumerate a regular grid over (mass_loading, calender_density, np_ratio)
    for each chemistry. Total evaluations = product(grid_resolution) × len(chemistries).
    """

    def run(self, config: SearchConfig) -> RunResult:
        grids = np.meshgrid(
            np.linspace(*config.bounds.mass_loading,   config.grid_resolution[0]),
            np.linspace(*config.bounds.calender_density, config.grid_resolution[1]),
            np.linspace(*config.bounds.np_ratio,        config.grid_resolution[2]),
            indexing="ij",
        )
        params_list = [...]   # flatten grid × chemistries
        results = self._evaluate_all(params_list, config)
        return self._build_run_result(results, config)
```

### MonteCarloOptimizer

```python
class MonteCarloOptimizer(BaseOptimizer):
    """
    Latin Hypercube Sampling over the 3D continuous space per chemistry.
    More efficient than a regular grid for the same number of evaluations.
    """

    def run(self, config: SearchConfig) -> RunResult:
        # Use scipy.stats.qmc.LatinHypercube for low-discrepancy sampling
        ...
```

### BayesianOptimizer (v1.2)

```python
class BayesianOptimizer(BaseOptimizer):
    """
    Sequential model-based optimization using a Gaussian Process surrogate.
    Acquisition function: Expected Hypervolume Improvement (multi-objective).
    Library: botorch (preferred) or scikit-optimize.
    """

    def run(self, config: SearchConfig) -> RunResult:
        # Iteratively: fit GP → select next point via acquisition → evaluate → update
        ...
```

---

## Pareto Extraction

Pure function — no side effects, no OpenCell dependency.

```python
def extract_pareto_front(
    df: pd.DataFrame,
    objectives: list[str],
    directions: list[Literal["max", "min"]],
) -> pd.DataFrame:
    """
    Return the subset of rows that are Pareto-optimal with respect to `objectives`.
    `directions` specifies whether each objective should be maximized or minimized.

    Default for this app:
        objectives  = ["specific_energy", "cost_per_energy"]
        directions  = ["max",             "min"]
    """
    ...
```

---

## Figure Builders

All functions return `go.Figure` objects (Plotly). They accept a `RunResult` or a `pd.DataFrame` — no simulator dependency.

```python
# figures.py

def pareto_scatter(
    result: RunResult,
    x: str = "specific_energy",
    y: str = "cost_per_energy",
    color_by: str = "chemistry",
    highlight_pareto: bool = True,
) -> go.Figure:
    """
    Scatter plot of all evaluated candidates.
    Pareto-optimal points are highlighted with a larger marker and connected
    by a stepped line to show the trade-off front.
    One color per chemistry.
    """
    ...

def pareto_by_chemistry(result: RunResult) -> go.Figure:
    """
    One Pareto front trace per chemistry, overlaid on a single axes.
    Non-Pareto candidates shown as faded scatter in the background.
    """
    ...

def parameter_heatmap(
    result: RunResult,
    chemistry: str,
    x_param: str = "mass_loading",
    y_param: str = "calender_density",
    z_metric: str = "specific_energy",
    np_ratio_slice: float | None = None,
) -> go.Figure:
    """
    2D heatmap of a metric across two parameters, holding the third fixed
    at `np_ratio_slice` (or the median if None). One figure per chemistry.
    """
    ...

def candidate_table(result: RunResult, n: int = 20) -> go.Figure:
    """
    Plotly Table of the top-N Pareto-optimal candidates with all parameters
    and metrics, sortable by any column.
    """
    ...

def cost_breakdown_sunburst(cell: ocd.CylindricalCell) -> go.Figure:
    """Thin wrapper around cell.plot_cost_breakdown() for a specific design."""
    return cell.plot_cost_breakdown()
```

---

## API Endpoints

Designed for Flask or FastAPI. The Dash app will call these same functions directly (no HTTP overhead) in development; HTTP routes are for a future decoupled deployment.

```
POST   /api/search
       Body: SearchConfig (JSON)
       Returns: { run_id, status: "running" }

GET    /api/search/{run_id}
       Returns: RunResult (JSON) when complete, or { status, progress } while running

GET    /api/search/{run_id}/figures/{fig_type}
       fig_type: pareto_scatter | pareto_by_chemistry | parameter_heatmap | candidate_table
       Returns: Plotly figure JSON (use plotly.io.to_json)

GET    /api/search/{run_id}/candidates
       Query params: chemistry, valid_only, pareto_only, limit, sort_by
       Returns: candidates as JSON (records orientation)
```

---

## Data Flow Summary

```
SearchConfig
    │
    ▼
BaseOptimizer.run()
    │
    ├── GridSearchOptimizer    → meshgrid over (ml, cd, np) × chemistries
    ├── MonteCarloOptimizer    → LHS samples per chemistry
    └── BayesianOptimizer      → GP + EHVI acquisition loop
    │
    ▼ list[dict]  (parameter combos)
CellSimulator.evaluate()       × N times
    │
    ▼ list[CellResult]
RunResult.candidates           (pd.DataFrame, all points)
    │
    ├── extract_pareto_front() → RunResult.pareto (pd.DataFrame)
    │
    └── figures.py
            ├── pareto_scatter()
            ├── pareto_by_chemistry()
            ├── parameter_heatmap()
            └── candidate_table()
```

---

## Implementation Order

1. `core/simulator.py` — get a single forward pass working end-to-end
2. `core/pareto.py` — test with synthetic data before wiring to simulator
3. `optimizers/grid_search.py` — run the MVP v1 grid
4. `results/figures.py` — `pareto_by_chemistry()` first (the key deliverable)
5. `optimizers/monte_carlo.py` — drop-in replacement for grid search
6. `api/routes.py` — wrap in HTTP for Dash integration
7. `optimizers/bayesian.py` — last, depends on BO library choice
