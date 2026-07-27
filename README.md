# EpiNoN-SEIRDQ

**Hierarchical network-of-networks epidemic simulator with GNN-based city risk prediction and a real-time decision support dashboard.**

EpiNoN-SEIRDQ models infectious disease spread across a two-level network-of-networks (NoN):

- **Level 2** — directed weighted city mobility graph (inter-city commuting, derived from OpenStreetMap)
- **Level 1** — per-city individual contact networks (households, workplaces, community, transport)
- **Cross-city mobility coupling blocks** form a supra-adjacency structure enabling multi-scale intervention
- **Stochastic SEIRDQ dynamics** with competing hazards (Susceptible → Exposed → Infectious → Recovered / Dead / Quarantined)
- **City risk labels** (low / warning / outbreak / critical), spectral risk, import-risk attribution
- **GNN-based city risk prediction** (optional, requires PyTorch)
- **Real-time alert engine** with four triggers: risk escalation, death threshold, peak-I capacity, import detection
- **Session-based authentication** for the web dashboard

Six disease presets span the full transmission ecology spectrum: COVID-19, Influenza, RSV, Measles, Mpox, and Hantavirus (zoonotic spillover).

The package ships with a bundled 12-city Calabria OSM network and a Flask-based **management web dashboard** with login, real-time alerts, counterfactual analysis, and decision support.

![System Architecture](docs/figures/fig_install_architecture.png)

---

## Installation Guide

### Prerequisites

- **Python >= 3.9**
- pip (or uv)
- (Optional, for GNN) PyTorch + PyTorch Geometric

### Step 1: Install the Package

![Installation Flow](docs/figures/fig_install_flow.png)

**From source (development):**

```bash
cd epinon_pkg
pip install -e .
```

**From the distributable archive:**

```bash
unzip epinon_seirdq_package.zip
cd epinon_pkg
pip install .
```

**With GNN support (optional):**

GNN training requires PyTorch and PyTorch Geometric, which have CUDA-specific install paths. Install them separately first:

```bash
# CPU-only (for testing)
pip install torch torch-geometric --index-url https://download.pytorch.org/whl/cpu

# Then install EpiNoN with the gnn extra
pip install -e ".[gnn]"
```

### Step 2: Verify Installation

```bash
epinon info
```

Expected output lists the six disease presets, nine policies, and the bundled OSM data path:

```
EpiNoN-SEIRDQ v1.0.0
==================================================
Bundled OSM data: /path/to/epinon_seirdq/data/calabria_osm
  Exists: True

Disease presets:
  covid         COVID-19      R0=    ~2.8  mode=Human-to-human
  influenza     Influenza     R0=    ~1.3  mode=Human-to-human
  rsv           RSV           R0=    >10*  mode=Human-to-human
  measles       Measles       R0=   12-18  mode=Airborne
  mpox          Mpox          R0=    ~1.8  mode=Close-contact
  hantavirus    Hantavirus    R0=     N/A  mode=Zoonotic

Policies:
  no_intervention         No Intervention
  uniform                 Uniform Controls
  ...
```

### Step 3: Start the Web Dashboard

![Dashboard Overview](docs/figures/fig_install_dashboard.png)

```bash
epinon dashboard --port 5000
```

Then open `http://127.0.0.1:5000` in your browser. You will be prompted to log in.

**Default credentials:** `admin` / `epinon2026`

![Login Screen](docs/figures/fig_install_login.png)

To change credentials, set environment variables before starting the server:

```bash
export EPINON_USER="your_username"
export EPINON_PASS="your_password"
epinon dashboard --port 5000
```

---

## Dashboard Guide

The dashboard has three tabs and a real-time alert system.

### Real-Time Alert System

![Alert System](docs/figures/fig_install_alerts.png)

The alert engine monitors the simulation and fires alerts when four conditions are met:

| Trigger | Description | Default Threshold |
|---------|-------------|-------------------|
| **Risk escalation** | A city's risk label increases (Low→Warning, Warning→Outbreak, Outbreak→Critical) | Automatic |
| **Death threshold** | Cumulative deaths cross a configurable threshold | 50 deaths |
| **Peak-I capacity** | Active infectious cases exceed a capacity threshold | 500 cases |
| **Import detection** | A new non-zero import-risk entry appears on a previously clean corridor | Automatic |

Alerts appear as:
- **Toast notifications** (bottom-right pop-ups, auto-dismiss after 5 seconds)
- **Alert bell badge** (header, shows count of unread alerts)
- **Alert panel** (slide-in from right, shows full timestamped log with severity colors)

Thresholds can be adjusted at runtime via the alert panel's configuration section.

### Live Simulation

![Live Simulation Workflow](docs/figures/fig_install_live_sim.png)

The Counterfactual Studio tab offers two simulation modes:

1. **Default (fast):** A lightweight JavaScript SEIR ODE solver runs in the browser for instant what-if analysis, using pre-computed metrics from the embedded results.
2. **Live (Python backend):** The "Run Live" buttons call the Flask `/api/simulate` endpoint, which runs the full EpiNoN-SEIRDQ stochastic simulator on the bundled Calabria OSM network. This is slower but uses the real network-of-networks dynamics and triggers the alert engine.

### Three Operational Tabs

1. **Observation Mode** — city risk map with color-coded GNN labels, epidemic curves, disease parameters, and outcome metrics
2. **Counterfactual Studio** — policy selection, parameter sliders with real-time re-simulation, side-by-side curve comparison, ranked multi-scenario table, and live simulation buttons
3. **Decision Support** — best-policy recommendation, corridor targeting advisor, border-closure timing chart, full six-disease comparison, and population-scaling limitations

---

## Quickstart

### Command-line interface

**Run a simulation:**

```bash
# COVID-19 with corridor-targeting policy, 6 Monte Carlo runs over 90 days
epinon simulate --disease covid --policy corridor_targeting --n-runs 6 --n-days 90

# Save full results (including epidemic curve) to JSON
epinon simulate --disease measles --policy city_isolation --n-runs 6 --curve -o results.json

# What-if: override transmission rate
epinon simulate --disease covid --policy corridor_targeting --beta 0.75 --beta-mob 0.35

# Corridor targeting with top-10 corridors
epinon simulate --disease mpox --policy corridor_targeting --top-k 10
```

**Start the web dashboard:**

```bash
epinon dashboard --port 5000
```

### Python API

```python
from epinon_seirdq.diseases import get_disease_config
from epinon_seirdq.simulator.adapter import OsmNoNAdapter
from epinon_seirdq.simulator.simulator import Simulator
from epinon_seirdq.simulator.controls import CorridorTargetingPolicy
from epinon_seirdq.alert_engine import AlertEngine

# Load a disease preset
cfg = get_disease_config("covid", seed=42, n_days=90)

# Use the bundled Calabria OSM network (12 cities)
import epinon_seirdq
from pathlib import Path
osm_dir = Path(epinon_seirdq.__file__).parent / "data" / "calabria_osm"

adapter = OsmNoNAdapter(osm_dir, cfg, pop_scale=0.01, pop_min=300, pop_max=1500)
policy = CorridorTargetingPolicy(cfg, top_k=5, corridor_retention=0.05)

sim = Simulator(cfg, policy=policy, generator=adapter)
history = sim.run()

# Run with alert engine
from epinon_seirdq.runner import run_experiment
result = run_experiment("covid", policy="corridor_targeting",
                        n_runs=3, n_days=90, collect_alerts=True)
print(f"Alerts: {len(result['alerts'])}")
print(f"Summary: {result['alert_summary']}")
```

### Running the web server programmatically

```python
from epinon_seirdq.web.app import create_app
app = create_app()
app.run(host="0.0.0.0", port=5000)
```

---

## Package structure

```
epinon_seirdq/
├── __init__.py              # Top-level: disease presets, version
├── cli.py                   # CLI entry point (epinon command)
├── diseases.py              # 6 disease config presets + metadata
├── runner.py                # Shared simulation runner (CLI + web + alerts)
├── alert_engine.py          # Real-time alert engine (4 triggers)
├── simulator/
│   ├── data_model.py        # SimulationConfig, City, Person, NoNState dataclasses
│   ├── dynamics.py          # SEIRDQDynamics: compute_foi, step, initialize
│   ├── generator.py         # NoNGenerator: synthetic NoN generation
│   ├── simulator.py         # Simulator: ties generator + dynamics + controls
│   ├── controls.py          # 9 control policies (NoIntervention ... BorderClosure)
│   ├── aggregation.py       # Aggregator, RiskLabeler, SpectralRisk, ImportRiskAttributor
│   ├── adapter.py           # OsmNoNAdapter: bridges OSM city graph → simulator
│   └── contact_provider.py  # ContactNetworkProvider ABC + SyntheticContactProvider
├── gnn/                     # GNN models (requires torch + torch-geometric)
│   ├── models.py            # L2GCN, L2GAT, NoNGCN, NoNGAT, SupraGCN, TemporalRollingGCN, ...
│   ├── dataset.py           # NoNDataset, build_static_dataset, build_temporal_dataset
│   ├── train.py             # train_model, evaluate_model, cross_validate
│   └── metrics.py           # compute_metrics, rank1_analysis, random_baseline_aupr
├── web/
│   ├── app.py               # Flask server with auth + alert endpoints
│   └── dashboard.html       # Management UI with login, alerts, 3 tabs
└── data/
    ├── calabria_osm/        # Bundled 12-city Calabria OSM network
    │   ├── cities_geocoded.csv
    │   ├── city_graph.graphml
    │   ├── coupling_matrix.csv
    │   ├── transition_matrix.csv
    │   └── ...
    ├── cities_calabria.csv
    └── results/
        └── dashboard_payload.json   # Pre-computed results embedded in dashboard
```

---

## Disease presets

| Disease | R0 | Incubation (d) | Infectious (d) | CFR (%) | Transmission mode |
|---------|-----|----------------|----------------|---------|-------------------|
| COVID-19 | ~2.8 | 5 | 6 | 0.6 | Human-to-human |
| Influenza | ~1.3 | 2 | 6 | 0.1 | Human-to-human |
| RSV | >10* | 5 | 7 | 0.2 | Human-to-human |
| Measles | 12–18 | 12 | 8 | 0.2 | Airborne |
| Mpox | ~1.8 | 7.6 | 8.3 | 0.3 | Close-contact |
| Hantavirus | N/A | 17 | 14 | 1.0 | Zoonotic spillover |

\* RSV intrinsic R0 > 10 (van Boven et al. 2020); effective susceptible fraction reduced by adult immunity.

## Control policies

| Policy | Description |
|--------|-------------|
| `no_intervention` | Baseline: no controls |
| `uniform` | Uniform distancing + testing across all cities |
| `threshold` | Label-triggered: controls scale with city risk level |
| `mobility_only` | Restrict inter-city mobility, no within-city controls |
| `centrality_targeting` | Target high-centrality individuals for distancing |
| `individual_only` | Within-city controls only, mobility untouched |
| `city_isolation` | Cut all inter-city mobility + within-city controls |
| `corridor_targeting` | Restrict only top-K highest-flow corridors (NoN-enabled) |
| `border_closure` | Close all borders at day T (timing experiment) |

---

## Web API endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | Yes | Dashboard HTML |
| GET | `/login` | No | Login form |
| POST | `/login` | No | Authenticate |
| GET | `/logout` | No | Clear session |
| GET | `/api/diseases` | Yes | Disease metadata + policy list |
| GET | `/api/payload` | Yes | Pre-computed dashboard payload |
| GET | `/api/health` | No | Health check |
| GET | `/api/alerts` | Yes | Current alert log + thresholds |
| POST | `/api/alerts/config` | Yes | Update alert thresholds |
| POST | `/api/simulate` | Yes | Run a simulation with alerts |
| POST | `/api/multi_policy` | Yes | Run all 5 policies for a disease |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `ModuleNotFoundError: No module named 'epinon_seirdq'` | Run `pip install -e .` from the `epinon_pkg` directory |
| `epinon: command not found` | Ensure the package is installed and your PATH includes the Python scripts directory |
| Dashboard shows "Authentication required" | Log in at `/login` with `admin` / `epinon2026` |
| `/api/simulate` returns 500 error | Check the Flask console output for the exception traceback |
| GNN import fails | Install PyTorch: `pip install torch torch-geometric` |
| `pdflatex: command not found` | Install TeX Live: `apt-get install texlive-latex-base texlive-latex-recommended` |

---

## Dependencies

**Core (required):** numpy, scipy, networkx, pandas, flask

**GNN (optional):** torch, torch-geometric

**Python:** >= 3.9

---

## License

MIT — see [LICENSE](LICENSE).

## Citation

If you use EpiNoN-SEIRDQ in your research, please cite:

```bibtex
@software{guzzi2026epinon,
  author = {Guzzi, Pietro Hiram},
  title = {{EpiNoN-SEIRDQ}: A Network-of-Networks Epidemic Framework},
  version = {1.0.0},
  year = {2026},
  doi = {10.5281/zenodo.XXXXXXX},
  url = {https://github.com/phguzzi/EpiNoN-SEIRDQ}
}
```

## Author

Pietro Hiram Guzzi — Magna Graecia University of Catanzaro — guzzi@unicz.it
