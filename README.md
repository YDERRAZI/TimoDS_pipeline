# TimoDS_pipeline

This repository contains the data generation pipeline for **TimoDS**, a 
comprehensive synthetically generated dataset of Timoshenko steel beam 
instances with general elastic supports and responses to elementary unit 
load cases.

TimoDS is introduced and described in full detail in the following soon to be published paper:

> Derrazi, Y. and Peluffo-Ordóñez, D.H. and Torres, J.C,  (2026). *TimoDS: A comprehensive and synthetically generated 
> dataset of Timoshenko steel beams with general elastic supports and 
> responses to elementary unit load cases*. Scientific Data.
> [Paper under review — DOI to be assigned upon publication]

---

## Dataset overview

TimoDS contains **60,000** beam instances distributed equally across six 
elementary unit load cases. Each instance corresponds to a fully defined 
linear static beam problem characterized by:

- A beam length L ∈ [1, 10] m
- A European steel cross-section from the IPE, HEA or HEB profile families
- Twelve elastic support stiffness parameters at both ends, spanning the 
  full continuum from free to quasi-clamped boundary conditions
- A relative load position α ∈ [0.05, 0.95]

For each instance, the dataset provides the full spatial distributions of:

- Kinematic quantities: ux, uy, uz, θx, θy, θz
- Internal forces: N, Vy, Vz, T, My, Mz
- Elastic strain energy: UD

All quantities are sampled at 21 equally spaced positions along the beam 
axis (ξ ∈ {0.00, 0.05, ..., 1.00}), yielding an output vector of 253 
quantities per instance.

The dataset is distributed as two CSV files joinable on a common 
identifier column:

| File | Description |
|------|-------------|
| `TimoDS_input_data.csv` | Sampling parameters: load case, section metadata, beam length, load position, dimensionless stiffness multipliers κ and ζ |
| `TimoDS_beam_results.csv` | Physical parameters and structural responses: section properties, physical stiffnesses, all sampled response quantities |

The dataset is publicly available at: https://doi.org/10.5281/zenodo.19324944
---

## Repository contents

```
TimoDS_pipeline/
├── README.md                        — this file
├── LICENSE                          — MIT License
├── TimoDS_pipeline_input_protocol.ipynb           — Jupyter notebook for input CSV generation
├── TimoDS_pipeline.gh               — Grasshopper pipeline definition
└── TimoDS_pipeline.png              — High-resolution overview of the pipeline
```

---

## Pipeline architecture

The data generation pipeline consists of two stages:

### Stage 1 — Input generation (Python)

The notebook `TimoDS_pipeline_input_protocol.ipynb` generates the input CSV file 
`TimoDS_input_data.csv` using Latin Hypercube Sampling (LHS) with a 
fixed random seed (`seed = 2`), guaranteeing exact reproducibility.

The notebook performs the following steps:

1. Draws 60,000 LHS samples in a 15-dimensional normalized design space
2. Maps each sample to physical parameters through bijective 
   transformations:
   - Beam length L: linear scaling onto a discrete grid 
     [1, 10] m with step ΔL = 0.05 m
   - Relative load position α: linear scaling onto a discrete 
     grid [0.05, 0.95] with step Δα = 0.05
   - Cross-section: index-based sampling from a catalogue of 66 
     European steel profiles (IPE, HEA, HEB)
   - Support stiffnesses: logarithmic sampling of dimensionless 
     multipliers κ and ζ over six decades
3. Assigns exactly 10,000 instances to each of the six elementary 
   load cases
4. Exports the result as `TimoDS_input_data.csv`

### Stage 2 — Structural analysis (Grasshopper/Karamba3D)

The Grasshopper definition `TimoDS_pipeline.gh` reads the input CSV 
and solves all 60,000 beam problems sequentially using the Karamba3D 
finite element plugin. For each instance, the pipeline:

1. Parses the input row and computes physical support stiffnesses from 
   the dimensionless multipliers
2. Assembles the beam model: geometry, cross-section, material, 
   boundary conditions, and unit load
3. Solves the linear static problem via Karamba3D's solver
4. Retrieves kinematic and internal force quantities at 21 equally 
   spaced positions along the beam axis
5. Buffers results in memory and flushes to disk in chunks of 5,000 
   instances

The pipeline is fully automated via a self-driving index driver 
component that iterates through all 60,000 input rows without manual 
intervention. A reset flag allows interrupted runs to be restarted 
cleanly.

---

## Requirements

### Python (Stage 1)

Python 3.8 or higher is required. Install dependencies with:

```bash
pip install -r requirements.txt
```

Dependencies:

```
numpy
scipy
pandas
matplotlib
seaborn
jupyter
```

### Grasshopper/Karamba3D (Stage 2)

The following software is required to run the Grasshopper pipeline:

| Software | Version tested |
|----------|---------------|
| Rhinoceros 3D | 7 or higher |
| Grasshopper | bundled with Rhino 7+ |
| Karamba3D | 2.2.0 or higher |

Karamba3D is a commercial plugin available at 
[https://www.karamba3d.com](https://www.karamba3d.com). A valid license 
is required.

---

## How to reproduce the dataset

### Step 1 — Generate the input CSV

1. Clone this repository:
```bash
git clone https://github.com/YDERRAZI/TimoDS_pipeline.git
cd TimoDS_pipeline
```

2. Install Python dependencies:
```bash
pip install -r requirements.txt
```

3. Open `TimoDS_pipeline_input_protocol.ipynb` in Jupyter:
```bash
jupyter notebook TimoDS_pipeline_input_protocol.ipynb
```

4. Edit the configuration cell at the top of the notebook to set your 
   desired output directory and filename:
```python
OUTPUT_DIR = "./outputs"
OUTPUT_FILENAME = "TimoDS_input_data.csv"
```

5. Run all cells. The output CSV will be saved to `OUTPUT_DIR`.

### Step 2 — Run the Grasshopper pipeline

1. Open Rhinoceros 3D and load `pipeline/TimoDS_pipeline.gh` in 
   Grasshopper

2. In the **CSV loader** component, update the input file path to point 
   to the CSV generated in Step 1

3. In the **result structuring** component, update the output file path 
   to your desired output location

4. Set `run = True` in the index driver component to start the pipeline. 
   The pipeline will iterate through all 60,000 instances autonomously 
   and write results incrementally to the output CSV in chunks of 5,000 
   instances

5. Once complete, set `run = False` and trigger a final `write` to flush 
   any remaining buffered results to disk

> **Note:** Running the full pipeline of 60,000 instances is 
> computationally intensive. On a standard workstation, expect a runtime 
> of several hours. The pipeline can be interrupted and resumed at any 
> time using the `start_i` and `end_i` parameters of the index driver 
> component.

---

## Dataset validation

The physical consistency of TimoDS has been verified through:

- **Equilibrium verification**: all force and moment equilibrium 
  residuals satisfy |ε| ≤ 2.66 × 10⁻¹⁵ kN(·m) across all 60,000 
  instances — at or below machine precision
- **Energy consistency**: the corrected strain energy UD_rebuilt, 
  recomputed from internal force fields via trapezoidal integration, 
  satisfies the work-energy theorem Wext = 2(UD + Uk) to high numerical 
  precision across the full dataset
- **Individual instance verification**: response profiles of 
  representative instances confirm the expected structural behaviour 
  including shear force discontinuities at load application points, 
  piecewise linear bending moment diagrams, and correct decoupling 
  between load cases

Full details are provided in the Technical Validation section of the 
companion paper.

---

## Citation

If you use TimoDS or this pipeline in your research, please cite:

```bibtex
@article{derrazi2026timods,
  author  = {Derrazi, Youssef and Peluffo-Ordóñez, D.H. and Torres, J.C},
  title   = {{TimoDS}: A comprehensive and synthetically generated dataset 
             of {Timoshenko} steel beams with general elastic supports and 
             responses to elementary unit load cases},
  journal = {Scientific Data},
  year    = {2026},
  note    = {Paper under review}
}
```

This entry will be updated with the DOI and volume/issue information 
upon publication.

---

## License

The code in this repository is licensed under the **MIT License** — 
see the `LICENSE` file for details.

The TimoDS dataset is licensed under **CC BY 4.0** 
(Creative Commons Attribution 4.0 International) — see 
[https://creativecommons.org/licenses/by/4.0/](https://creativecommons.org/licenses/by/4.0/).

---

## Contact

Youssef Derrazi

youssef.derrazi@sdas-group.com

Youssef.DERRAZI@um6p.ma

SDAS Research Group

Universidad de Granada - UGR

University Mohammed VI Polytechnic - UM6P
