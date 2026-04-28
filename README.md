# PD-FHC: Reference Implementation

Reference Python implementation accompanying the paper *Plausible Deniability in Fully Homomorphic Computation* (PD-FHC). The script demonstrates the full protocol on four Boolean circuits, runs benchmarks across configurations, and produces the plots and JSON used in the evaluation section.

This is research code. It is single-file, single-threaded, and optimised for clarity over speed. It is not production-ready and is not meant to be.

---

## What the script does

Running `pdfhc_reference.py` executes two parts in sequence:

**Part 1 — Verbose walkthrough.** Runs the Small Boolean circuit `(A AND C) OR (NOT A AND B) OR (NOT B AND NOT C)` with `L = 4` scenarios (1 real, 3 decoys). Prints every step: which cover image is bound to which wire, the secret position `(row, col, channel)` for each scenario in plain English, every Fredkin gate applied, and the extracted output. Ends with a coercion vignette in which Alice reveals decoy 1.

**Part 2 — Benchmarks.** Silently runs 10 configurations × 20 repetitions each, then prints two summary tables (TFHE comparison, multi-location overhead) and writes:

- `benchmark_plots/compute_time_by_circuit.pdf`
- `benchmark_plots/tfhe_comparison.pdf`
- `benchmark_plots/multilocation_overhead.pdf`
- `benchmark_plots/throughput.pdf`
- `benchmark_plots/latency_breakdown.pdf`
- `pdfhc_benchmarks.json` (raw timings)

---

## Circuits implemented

| Name        | Gates | Inputs (ℓ) | Outputs | Description                                  |
|-------------|------:|-----------:|--------:|----------------------------------------------|
| Threshold-3 |     5 |          3 |       1 | Majority: output 1 iff ≥2 of {A,B,C} are 1   |
| Small-Bool  |    14 |          3 |       1 | (A∧C) ∨ (¬A∧B) ∨ (¬B∧¬C)                     |
| 4-bit-Adder |    88 |          8 |       5 | Full 4-bit binary addition                   |
| 8-bit-Mult  |   302 |         16 |      16 | 8-bit multiplication (target ~289, lands at 302) |

The adder and multiplier are generated programmatically by `_gen_adder_circuit()` and `_gen_mult_circuit(289)`. Gate counts above are what the script actually produces, not the rounded targets in earlier drafts.

---

## Requirements

- Python 3.8+
- `numpy`
- `Pillow` (optional but strongly recommended; without it the script falls back to synthetic random images)
- `matplotlib`

```bash
pip install numpy Pillow matplotlib
```

The script tries matplotlib backends in order: `TkAgg` → `Qt5Agg` → `Agg`. If only `Agg` is available, plots are still saved to disk but no interactive windows appear. Set `DISPLAY_IMAGES = False` near the top of the file if you don't want any windows.

---

## Cover images

The script expects 128×128 PNG cover images in `C:/cover` (Windows path, hardcoded as `COVER_IMAGE_DIR`). Change this constant at the top of the file for other systems:

```python
COVER_IMAGE_DIR = "C:/cover"   # change to e.g. "/home/you/covers" on Linux
```

Behaviour:

- If the directory exists and contains PNGs, the first 200 are loaded (sorted by filename). Non-128×128 images are resized via Lanczos.
- If the directory is missing or Pillow is not installed, the script generates 200 synthetic random `uint8` images and prints a warning. Benchmarks remain valid; only the PSNR/SSIM numbers become uninformative because there is no semantic structure to preserve.
- If you have fewer than 200 PNGs, images are reused with a `(reuse)` suffix in their printed names. Reuse across wires is harmless for protocol correctness but means PSNR is averaged over fewer distinct covers.

---

## Running

```bash
python pdfhc_reference.py
```

Total runtime on the reference hardware (HP EliteBook 840 G8, i5-1135G7, 16 GB) is roughly 3–5 minutes for the full benchmark sweep.

If `DISPLAY_IMAGES = True`, the script blocks on `input()` after Part 1 to let you inspect the wire-to-image grid. Hit Enter to proceed to benchmarking.

---

## Configuration knobs

All near the top of the file:

| Constant            | Default       | Effect                                                       |
|---------------------|---------------|--------------------------------------------------------------|
| `COVER_IMAGE_DIR`   | `"C:/cover"`  | Directory of PNG covers                                      |
| `DISPLAY_IMAGES`    | `True`        | Show matplotlib windows during walkthrough                   |

Inside `main()`:

- `H, W = 128, 128` — image size for Part 1
- `bench_configs` — the list of `(circuit, H, W, L)` tuples Part 2 sweeps
- The `benchmark(..., num_runs=20)` call controls per-configuration repetitions

---

## Output format

`pdfhc_benchmarks.json` is a JSON object with one entry per benchmark configuration:

```json
{
  "benchmarks": [
    {
      "circuit_name": "Small-Bool",
      "gates": 14,
      "image_size": "128x128",
      "H": 128, "W": 128, "L": 4,
      "num_wires": 49,
      "embed_ms_mean": 0.42, "embed_ms_std": 0.05,
      "compute_ms_mean": 8.70, "compute_ms_std": 0.31,
      "extract_ms_mean": 0.01, "extract_ms_std": 0.00,
      "total_ms_mean": 9.13, "total_ms_std": 0.32,
      "per_gate_ms": 0.62,
      "throughput": 1609.2,
      "comm_mb": 2.3,
      "psnr": 51.14,
      "num_runs": 20
    }
  ]
}
```

Per-run gate-level timings are kept in memory during execution but stripped from the JSON to keep file size reasonable.

---

## Reproducing paper numbers

The numbers in the paper's evaluation tables come directly from this script's output. Sources of variation between runs:

- Position generation uses `secrets.token_bytes` (cryptographically random) — every run picks different secret positions.
- Cover image selection is sorted but depends on what's in `C:/cover`.
- Timing variation is hardware-dependent; the paper reports means over 100 runs, this script defaults to 20.

Bump `num_runs=20` to `num_runs=100` in the `benchmark()` call inside `main()` to match the paper's averaging protocol exactly.

---

## What the code does not do

Honest list of things missing from this reference implementation:

- **No GPU path.** The Fredkin gate is vectorised with NumPy only. The 10–50× speedup mentioned in the paper is projected from CuPy benchmarks elsewhere, not measured here.
- **No verifiable computation.** Carol is assumed to execute faithfully; there's no integrity check on her output beyond the protocol's own correctness.
- **No image reuse optimisation.** Each wire gets its own image array. The `O(m · h · w)` storage discussed in the paper's limitations section is the actual storage footprint of this implementation.
- **No formal circuit obfuscation.** Wire renaming and dummy-gate insertion described in the paper are *not* applied in this reference — the script uses the real circuit specs directly. Reviewers asking to inspect obfuscation behaviour will find that path stubbed out.
- **No constant-time guarantees beyond what NumPy provides.** Side-channel claims in the paper apply to the abstract protocol; the implementation makes no effort to harden against micro-architectural attacks.
- **Single-threaded.** All measurements assume one core.

---

## File layout

```
pdfhc_reference.py       # the entire implementation (~1,250 lines)
pdfhc_benchmarks.json    # produced on run
benchmark_plots/         # produced on run, contains 5 PDFs
C:/cover/                # you provide; 128x128 PNGs
```

---

## Citation

If you build on this code, cite the PD-FHC paper. Bibliographic details will be filled in once the paper is accepted; for now the artifact is anonymous.

For the steganographic-computation primitive this work builds on, see ProSt (Ahmad and Rass, *IEEE Access*, 2026, DOI 10.1109/ACCESS.2026.3656995).
