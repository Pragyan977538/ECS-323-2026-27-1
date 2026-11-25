# Shared control for a brain-controlled wheelchair

**Pragyan Mohanty — roll 24250**

A person imagines a movement (left hand, right hand, feet or tongue) and an EEG decoder turns it into a wheelchair command: turn left, turn right, go forward or stop. The decoder is wrong more than half the time and takes seconds to decide, so a chair that simply obeys it crashes most of the time. Instead of chasing a better decoder, this project treats it as a fixed, imperfect sensor and asks a control question: **how should control be shared between the user and an autopilot so the chair still reaches its goal safely?**

### Objectives

1. **Measure the decoder honestly.** Train a CSP + LDA decoder on BCI Competition IV dataset 2a (9 subjects, trained on day 1, tested on day 2) and measure what matters for control: how often it is wrong, which way it is wrong, how confident it is, and how long it takes to decide.
2. **Turn those measurements into a model of the command channel**, so the simulated wheelchair receives commands with each subject's real errors, delays and confidence.
3. **Compare ways of sharing control.** The motors get `u = α·u_user + (1 − α)·u_auto`; six rules for choosing the user's share α are compared over thousands of paired simulated drives. The proposed rule, `α = confidence × (1 − risk)` with risk from the time to collision, reaches the goal in 99.6% of drives.
4. **Find the stability limit under delay.** Derive exactly how much loop delay the shared loop can tolerate for each α, and check the result by simulation.

### Where the control theory is

The two notebooks are mostly data science and simulation: decoding EEG, modelling the decoder, and running and analysing the wheelchair experiments. The core control-theory work is the **stability derivation** in `report/math_derivation.pdf` (typeset) and `report/math_derivation_handwritten.pdf` (my handwritten version). It covers:

- a linear model of the shared loop: a damped double integrator, an LQR autopilot (gains derived by hand from the Riccati equation) and a delayed PD user;
- the characteristic equation with the delay term `e^(−τs)`, and why Routh–Hurwitz and root locus cannot be used on it;
- the exact delay margin `τ_max(α)`, by finding where a root can cross the imaginary axis (the delay drops out and a quadratic is left);
- the range of α that is stable for any delay (α below about 0.44), and the same analysis for the wheelchair's heading loop;
- checks against a Padé approximation and direct simulation of the delay equation.

The main report, `report/report.pdf`, presents the whole project and all results.

---

## 1. Folder map

```
pragyan_24250/
├── README.md                  this file
├── requirements.txt           exact package versions
├── src/
│   ├── part1_decoder.ipynb    Part 1 (run first)
│   ├── part2_shared_control.ipynb   Part 2 (run second)
│   ├── evaluate.py            Part 1 numbers without Jupyter (optional)
│   └── decoder/               decoder code used by Part 1
│       ├── data.py            loads BCI IV 2a through MOABB, 8–30 Hz filter, epochs
│       ├── pipeline.py        CSP + LDA, Platt calibration, run_subject()
│       └── metrics.py         ECE, reliability curve, decision latency
├── logs/
│   ├── decoder_models/        trained decoder for each subject (subject_01.joblib … subject_09.joblib)
│   ├── part1_run.log          Part 1 summary numbers
│   └── part2_run.log          Part 2 headline numbers
├── results/
│   ├── part1/
│   │   ├── summary_by_subject.csv   accuracy, kappa, ECE, latency per subject
│   │   ├── decoder_traces.csv       one row per test trial (input to Part 2)
│   │   └── figures/                 9 figures (incl. raw EEG)
│   └── part2/
│       ├── episodes.csv             log of all 32,400 simulated drives (input to Part 2)
│       ├── figures/                 17 figures (01_… to 17_…)
│       └── tables/                  11 CSV tables behind the report's results
└── report/
    ├── report.pdf             the main report: all results
    ├── math_derivation.pdf    the stability derivations, step by step
    ├── math_derivation_handwritten.pdf   handwritten version of the derivation
    └── report.tex, math_derivation.tex   LaTeX sources (figures are read from results/)
```

`logs/` and `results/` already hold the outputs of our run; running the notebooks again regenerates them.

---

## 2. Setup and run

**Dataset.** [BCI Competition IV dataset 2a](https://www.bbci.de/competition/iv/) (Graz motor imagery: 9 subjects, 4 classes, 22 EEG channels at 250 Hz, two sessions on different days). Description: [desc_2a.pdf](https://www.bbci.de/competition/iv/desc_2a.pdf). No manual download is needed: Part 1 fetches it through MOABB as [`BNCI2014_001`](https://neurotechx.github.io/moabb/generated/moabb.datasets.BNCI2014_001.html) on the first run, about 780 MB (18 `.mat` files) into `~/mne_data`; later runs reuse it.

Python 3.12 or 3.13. From this folder:

```bash
python -m venv .venv
.venv\Scripts\activate          # macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
```

Run the notebooks in order (Part 2 uses files written by Part 1):

```bash
jupyter nbconvert --to notebook --execute --inplace src/part1_decoder.ipynb --ExecutePreprocessor.timeout=-1
jupyter nbconvert --to notebook --execute --inplace src/part2_shared_control.ipynb --ExecutePreprocessor.timeout=-1
```

The notebooks can also be opened in Jupyter and run top to bottom; they find the project folder on their own. `python -m src.evaluate` reproduces Part 1's per-subject numbers without Jupyter.

---

## 3. Results

The run logs (`logs/part1_run.log`, `logs/part2_run.log`) collect the main numbers.

**Part 1: decoder (report Section 3, Table 3)**

| Quantity | Value | File |
|---|---|---|
| Mean calibrated accuracy (chance = 0.25) | 0.443 | `results/part1/summary_by_subject.csv` |
| Mean Cohen's kappa | 0.258 | same |
| Mean ECE, calibrated | 0.096 | same |
| Mean ECE, raw (two ways of scoring it; both in Table 3) | 0.162 / 0.196 | `summary_by_subject.csv` / `logs/part1_run.log` |

**Part 2: shared control (report Section 5)**

| Quantity | Value | File | Report |
|---|---|---|---|
| Success, user only | 0.119 | `tables/headline_main.csv` | Table 5 |
| Success, autopilot only | 1.000 | same | same |
| Success, adaptive (risk-gated) | 0.996 | same | same |
| Mean authority given to the user, adaptive | 0.118 | same | same |
| Adaptive minus user only, subject-level 95% CI | 0.878 [0.791, 0.957] | `tables/paired_subject_level.csv` | Table 6 |
| Subjects with significant error bursts | 0 of 9 | `logs/part2_run.log` | Section 4.1 |
| Pooled confidence AUC | 0.676 | `logs/part2_run.log` | Section 4.1 |

**Stability (report Section 6, Table 8; derivation in `math_derivation.pdf`)**

| Quantity | Value | File |
|---|---|---|
| LQR gains K1, K2 | 3.162, 1.885 | `logs/part2_run.log` |
| τ_max at α = 1 | 0.529 s | `tables/tau_max_formula_vs_simulation.csv` |
| Delay-independent below α* | 0.441 | `logs/part2_run.log` |
| Heading loop: α*, τ_max(1) | 0.613, 0.647 s | `logs/part2_run.log` |
| Formula vs direct DDE simulation | agree to better than 1e-5 (relative) | `tables/tau_max_formula_vs_simulation.csv` |

All figures are in `results/part1/figures/` and `results/part2/figures/`; Part 2 file names carry the notebook section they come from (e.g. `14_s6_…` is from Section 6).

---

## 4. Notes

- **Saved drive log.** The full set of experiments is 32,400 simulated drives, saved in `results/part2/episodes.csv`. Part 2 analyses that log and, as a check, re-simulates 72 drives from scratch (notebook Section 6.4); they match the log to about one part in 10⁹.
- **Two raw-ECE values.** 0.162 scores the uncalibrated confidence against the raw prediction, 0.196 against the calibrated prediction (the decision the wheelchair uses). Both are printed in Part 1 and shown in the report.
- **Reproducibility.** Seeds are fixed (decoder 20260901, simulation 0–19). Decoder numbers match exactly; continuous simulation values can differ in the 9th significant digit on another machine, which does not change any reported number.
- **PDFs** can be rebuilt with `pdflatex` in `report/` (run twice per file); figures are read directly from `results/`.
