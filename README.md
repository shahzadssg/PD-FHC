# PD-FHC: Plausible Deniability in Fully Homomorphic Computation

Reference implementation and steganalysis evaluation for the paper *Plausible
Deniability in Fully Homomorphic Computation* (PD-FHC). PD-FHC hides both the
existence and the intent of an outsourced computation by embedding it in the
least-significant-bit (LSB) plane of cover images and running it as a pixel-wise
Fredkin-gate circuit on an untrusted cloud.

This is research code. It is single-file, single-threaded, and written for
clarity over speed. It is not production ready and is not meant to be.

---

## What this is, and what it is not

**It is:**
- A working end-to-end prototype of the PD-FHC protocol on four Boolean circuits.
- A benchmark harness that reproduces the paper's evaluation numbers.
- A steganalysis evaluation that tests whether the stego medium `M'` is
  distinguishable from a legitimate input to the declared cloud service.

**It is not:**
- A security guarantee for an arbitrary cover. The indistinguishability claim
  holds only for cover services whose legitimate inputs already carry a
  high-entropy LSB plane. See [Security scope](#security-scope).
- A proof of resistance to modern learned steganalysis. The bundled detectors
  (RS analysis, chi-square) are classical. They are a floor, not a bar.

---

## Repository layout

```
PD-FHC.ipynb          # full protocol + benchmarks (single file)
pdfhc_benchmarks.json       # produced by a benchmark run
benchmark_plots/            # produced by a benchmark run (5 PDFs)
C:/cover/                   # you provide: 128x128 PNG covers
```

---

## Requirements

- Python 3.8+
- `numpy`
- `Pillow` (recommended; without it the reference falls back to synthetic images)
- `matplotlib` (reference benchmarks only)

```bash
pip install numpy Pillow matplotlib
```

---

## Part 1: the reference implementation

Running PD-FHC.ipynb` executes two parts in sequence.

**Verbose walkthrough.** Runs the Small Boolean circuit
`(A AND C) OR (NOT A AND B) OR (NOT B AND NOT C)` with `L = 4` scenarios
(1 real, 3 decoys). It prints every step: which cover image is bound to which
wire, the secret position `(row, col, channel)` for each scenario, every Fredkin
gate applied, and the extracted output. It ends with a coercion vignette in
which Alice reveals decoy 1.

**Benchmarks.** Runs 10 configurations with 20 repetitions each, prints the TFHE
comparison and multi-location overhead tables, and writes the plots and
`pdfhc_benchmarks.json`.

```bash
PD-FHC.ipynb
```

### Circuits

| Name        | Gates | Inputs (l) | Outputs | Description                                |
|-------------|------:|-----------:|--------:|--------------------------------------------|
| Threshold-3 |     5 |          3 |       1 | Majority: 1 iff at least 2 of {A,B,C} are 1 |
| Small-Bool  |    14 |          3 |       1 | (A AND C) OR (NOT A AND B) OR (NOT B AND NOT C) |
| 4-bit-Adder |    88 |          8 |       5 | Full 4-bit binary addition                 |
| 8-bit-Mult  |   302 |         16 |      16 | 8-bit multiplication                       |

Gate counts are what the code actually produces. The adder and multiplier are
generated programmatically.

### Cover images

The reference expects 128x128 PNG covers in `COVER_IMAGE_DIR` (default
`C:/cover`, a hardcoded Windows path). Change the constant at the top of the
file for other systems.

- Real PNGs present: the first 200 are loaded (sorted by filename), resized if
  needed.
- Directory missing or no Pillow: 200 synthetic random images are generated and
  a warning is printed. Benchmarks stay valid, but PSNR and any steganalysis
  number become meaningless because there is no structure to preserve. Use real
  covers.

---

## Part 2: steganalysis evaluation

`PD-FHC.ipynb` measures whether `M'` is distinguishable from a
legitimate input to the declared service. It implements RS analysis (Fridrich,
Goljan, Du, 2001) and the chi-square PoV attack (Westfeld, Pfitzmann, 1999), and
reports a permutation-test p-value rather than a bare threshold.

Open it in Jupyter and run **Kernel > Restart & Run All**. Set `COVER_IMAGE_DIR`
in the Step 2 cell to your real covers. Running only one cell can leave a stale
function in memory, which is the most common way to get a wrong number here.

It runs two experiments:

| Experiment | Clean baseline | Meaning | Expected |
|---|---|---|---|
| 1 | untouched natural photo | the broken cover story | Delta ~1, p small, SEPARABLE |
| 2 | legitimate input of an LSB-randomising service | the actual claim | Delta ~0, p > 0.05, indistinguishable |

**Read the p-value, not the point estimate.** `p > 0.05` means the detector
cannot separate the two populations beyond chance. The Step 1 cell prints a null
calibration that must come back not significant, which is the built-in proof
that the estimator is not manufacturing separability.

### How to interpret the result, honestly

A passing Experiment 2 (`p > 0.05`) shows that, against these two classical
detectors, `M'` is not separable from a uniform-LSB legitimate input. It does
**not** show that PD-FHC resists steganalysis. Two reasons:

1. **The bundled detectors are weak.** RS and chi-square are from 1999 and 2001.
   Modern detectors (ensemble classifiers on rich models, or CNNs such as SRNet)
   are far stronger. A near-perfect classical result is routinely overturned by
   a learned one.
2. **Experiment 2 uses an idealised baseline.** Its clean population has a
   perfectly uniform CSPRNG LSB plane, so the stego and the baseline differ in
   only a handful of bits and are nearly the same distribution by construction.
   No detector separates two samples from one distribution. The number you get
   is close to tautological.

The experiment that actually matters, and the one a top-tier reviewer will
demand, is `M'` against **real sample outputs of the specific service you claim
as cover**, scored by a learned detector with a held-out test set and a
significance check. Real dithering or sensor-noise pipelines leave LSB
statistics that are high-entropy but not perfectly uniform, and any detectable
gap lives in that difference. Comparing only against a CSPRNG-uniform baseline
will return `p = 1.0` forever and teach you nothing.

---

## Security scope

State precisely what is established and what is assumed.

**Proven unconditionally.** Extraction correctness. Hamming-weight preservation
of Fredkin gates. That a uniform LSB plane maps to a uniform LSB plane under any
Fredkin circuit. The reduction from multi-location privacy to single-position
hiding.

**Proven, conditional on a stated assumption.** Information-theoretic position
privacy and existence hiding for the image instantiation hold given the
high-entropy LSB-plane condition. The guarantee degrades to a
statistical-distance bound when the condition holds only approximately. There is
no super-exponential or factorial advantage. Earlier drafts claimed one. It was
a counting error and has been removed.

**Engineering heuristics, not proven.** Circuit obfuscation raises reverse
engineering cost but gives no function-hiding guarantee. Intent hiding rests on
the semantic plausibility of decoy circuits, which is currently manual and an
open problem to automate. Side-channel freedom is argued under an idealised
constant-time model.

**Out of model.** Collusion between the cloud and the coercer. Active deviation
by the cloud. Steganalysis of the cover medium against a learned detector.

The headline restriction: **PD-FHC does not hide computation inside an arbitrary
photograph.** It applies to cover services whose legitimate inputs already carry
a high-entropy LSB plane (dithering, LSB-noise injection, watermark robustness
testing, sensor-noise or encrypted-domain imagery). Content-adaptive embedding
cannot recover the natural-photo case, because uniform per-pixel computation
destroys the distortion minimisation those schemes depend on.

---

## Known limitations of the code

- **No GPU path.** The Fredkin gate is vectorised with NumPy only. The speedups
  mentioned in the paper are projected from CuPy benchmarks elsewhere, not
  measured here.
- **No verifiable computation.** The cloud is assumed to execute faithfully.
- **No image-reuse optimisation.** Each wire gets its own image array, so the
  storage footprint is the full `O(m * h * w)`.
- **No formal circuit obfuscation.** Wire renaming and dummy-gate insertion are
  described in the paper but stubbed out here. The reference uses the real
  circuit specs.
- **No constant-time guarantees** beyond what NumPy provides.
- **Single-threaded.** All measurements assume one core.
- **Steganalysis is classical only.** A learned detector against real
  declared-service inputs is required before any indistinguishability claim, and
  is not included.

A note on methodology, since it bit this project once: any detector accuracy
must come from a held-out test set with a permutation or similar significance
test. Choosing a threshold on the same data you score on overfits and reports a
spurious positive even on identical populations. This is harmless to forget with
a weak detector and dangerous with a high-capacity one.

---

## Reproducing the paper numbers

The evaluation tables come from PD-FHC.ipynb`. Sources of run-to-run
variation: secret positions use `secrets.token_bytes`, cover selection depends
on your `C:/cover` contents, and timings are hardware dependent. The paper
averages over 100 runs. Bump `num_runs=20` to `num_runs=100` in the
`benchmark()` call to match.

---

## Citation

Citation details for the PD-FHC paper will be added once it is accepted.

For the steganographic-computation primitive this work builds on, see ProSt
(Ahmad and Rass, *IEEE Access*, 2026, DOI 10.1109/ACCESS.2026.3656995).
