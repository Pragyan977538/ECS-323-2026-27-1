# Source Directory (`src/`)

Code for both parts of the project. The two notebooks are the main entry points; everything in `results/` and `logs/` is produced by them.

## Directory Contents

- **`part1_decoder.ipynb`**: Part 1, EEG motor-imagery decoding. Loads BCI IV 2a, trains CSP + shrinkage LDA per subject, applies Platt calibration, and produces the accuracy, reliability/ECE and latency figures (`results/part1/`, models in `logs/decoder_models/`).
- **`part2_shared_control.ipynb`**: Part 2, the closed-loop shared-control wheelchair. Contains the error channel, wheelchair model, simulated user, DWA autopilot, safety filter, arbitration rules and the stability checks (`results/part2/`).
- **`decoder/`**: Helper module used by Part 1 only (data loading, CSP + LDA pipeline, calibration and metrics). The Part 2 code lives inside its notebook.
- **`evaluate.py`**: Runs the Part 1 decoder for all 9 subjects without the notebook and writes `results/part1/summary_by_subject.csv`.
- **`__init__.py`**: Makes `src` a package so the imports above work.

## Usage

From the repository root:

```bash
jupyter notebook          # then open src/part1_decoder.ipynb and src/part2_shared_control.ipynb
python -m src.evaluate    # optional: Part 1 summary only
```

Run `evaluate.py` with `python -m` from the root as shown; `python src/evaluate.py` will not find the `src` package.
