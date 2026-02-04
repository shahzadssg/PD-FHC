

## Abstract

> Outsourcing sensitive computations to untrusted cloud providers introduces two primary challenges: ensuring the confidentiality of inputs and outputs, and concealing the true purpose of the computation. Fully Homomorphic Encryption (FHE) facilitates encrypted data processing, while Trusted Execution Environments (TEEs) provide secure hardware enclaves that protect data confidentiality and integrity. However, both approaches lack plausible deniability, which is the ability to credibly deny the genuine intent of a computation under coercion. To address this limitation, Plausible Deniability in Fully Homomorphic Computation (PD-FHC) is proposed as a lightweight framework. PD-FHC advances homomorphic steganography by embedding computations within innocuous data, thereby supporting user-level deniability in the presence of coercive adversaries.
>
> PD-FHC enables a client, referred to as Alice, to embed multiple plausible computation scenarios, including one authentic computation and several decoy computations. These scenarios are concealed at secret locations within innocuous images. An obfuscated universal Boolean circuit, constructed from Fredkin gates, is also utilized to evaluate any Boolean function. Both the images and the circuit are transmitted to an honest-but-curious cloud provider, Carol, who processes all computations identically and in parallel, rendering them indistinguishable. In the event of coercion, Alice can disclose a harmless decoy computation and its verifiable outcome to the coercer.
>
> The security analysis adopts a two-phase threat model that differentiates between the honest-but-curious behavior of the cloud provider and the active demands of a coercive adversary. Based on this distinction, novel coercion games are formulated, and a proof is provided demonstrating that adversaries possess only a negligible advantage in distinguishing genuine computations from decoys. To validate feasibility for moderate-sized circuits, a proof-of-concept Python implementation employing vectorized operations is presented. GPU acceleration is identified as a key optimization for efficiency and scalability. Consequently, PD-FHC constitutes a hardware-agnostic and computationally efficient alternative to FHE, and represents the first system to unify computational privacy, existence hiding, and robust intent deniability for coercion resistance.

## Introduction

In an era where cloud computing is ubiquitous, protecting sensitive data and the very intent behind computations remains a critical challenge. While established techniques like FHE and TEEs offer robust data confidentiality, they fall short when facing a coercive adversary who demands to know the *true purpose* of a computation. PD-FHC addresses this gap by introducing a novel framework that not only encrypts and hides computations but also provides **plausible deniability** – the ability for a user to credibly deny the actual intent of their cloud workload.

This project presents a proof-of-concept implementation of PD-FHC, demonstrating its core mechanisms, evaluating its performance, and illustrating its coercion resistance properties.

## Features

- **Plausible Deniability**: Alice can embed multiple "plausible" computations (one real, many decoys) within cover images.
- **Homomorphic Steganography**: Computations are performed directly on stego-images, preventing the cloud provider (Carol) from distinguishing real from decoy operations.
- **Universal Boolean Circuit**: Utilizes Fredkin gates to construct any arbitrary Boolean function.
- **Circuit Obfuscation**: Random renaming of wires and insertion of dummy gates/ancillaries to hide the original circuit structure from Carol.
- **Coercion Resistance Protocol**: Alice can reveal a verifiable decoy computation to a coercer (Eve) without compromising her real secret.
- **Vectorized Operations**: Optimized `numpy` implementations for significant speedup in pixel-wise gate application.
- **Comprehensive Benchmarking Suite**: Includes tools for measuring:
    - Throughput scaling with image size.
    - Multi-location overhead.
    - Per-gate latency.
    - Scalability analysis (execution time vs. pixel count).
    - Baseline comparisons (vectorized vs. naive approaches).
    - Overhead percentage calculations.
- **Publication-Quality Plot Generation**: Automatically generates visual analyses of benchmark results.

## Threat Model and Security

Our security analysis operates under a two-phase threat model:

1.  **Honest-but-Curious Cloud Provider (Carol)**: Carol executes the provided obfuscated circuit on the stego-images. She is honest in her execution (doesn't maliciously alter results) but curious, meaning she attempts to learn as much as possible about Alice's computation. PD-FHC ensures Carol cannot distinguish between real and decoy computations, nor can she discern the actual function being computed due to obfuscation and random LSBs.

2.  **Coercive Adversary (Eve)**: Eve is an active adversary who can coerce Alice to reveal the details of *one* of her computations. Alice, under coercion, reveals a carefully chosen decoy computation. PD-FHC's security relies on the fact that Eve, even with full knowledge of the revealed decoy's inputs, outputs, and the entire protocol, cannot prove that this revealed computation is not Alice's true intention. The cryptographic randomization of LSBs across the entire image, combined with the indistinguishable processing of multiple locations, makes all computations appear equally plausible to Eve.

We formalize these guarantees through novel "coercion games" and prove that adversaries gain only a negligible advantage in distinguishing genuine computations from decoys.

## Technical Details

### Fredkin Gate

The Fredkin gate is a universal reversible logic gate. It is a 3-input, 3-output gate where the first input (control, C) determines whether the other two inputs (X, Y) are swapped at the output. If C=0, outputs are (C, X, Y). If C=1, outputs are (C, Y, X). Its conservative property (number of 1s in input equals number of 1s in output) makes it useful in reversible computing paradigms.

In PD-FHC, Fredkin gates are applied homomorphically across image channels, where each pixel's LSB represents a bit in the computation.

### Homomorphic Steganography

This technique allows computations to be performed directly on data hidden within innocuous cover images. The least significant bit (LSB) of each pixel (or a specific channel of a pixel) is used to embed secret bits. The "homomorphic" property means that operations performed on the stego-image (like applying a Fredkin gate pixel-wise) directly correspond to operations on the embedded secret bits, without needing to extract them first. Critically, PD-FHC randomizes *all* LSBs of the image, making the embedded secret indistinguishable from random noise to an observer without Alice's knowledge of the secret locations.

### Circuit Obfuscation

To prevent Carol from analyzing the circuit statically and potentially identifying critical paths or inputs, Alice obfuscates her circuit by:
1.  **Renaming**: All original wire names are replaced with cryptographically random, opaque identifiers.
2.  **Dummy Ancillaries**: Additional ancillary wires with random bit values are introduced.
3.  **Dummy Gates**: Irrelevant Fredkin gates operating on these dummy wires (or other random available wires) are added, increasing the circuit's complexity and hiding the real computation graph.

### Coercion Resistance Protocol

When Alice faces coercion, she does not reveal her *real* computation. Instead, she selects one of the *decoy* computations she prepared. She then provides the coercer (Eve) with:
-   The exact input values for the decoy.
-   The expected output values for the decoy.
-   The precise pixel/channel location where this decoy computation occurred.
-   A plausible "cover story" for why she was running that specific computation.

Eve can then re-simulate this specific scalar computation (using the original circuit logic) and verify that Alice's claimed inputs correctly lead to the claimed outputs at the specified location. However, because all locations (real and decoys) were processed identically and the LSBs of all images are randomized, Eve cannot prove that the revealed decoy was not Alice's true intention.

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for demonstration and development purposes.

### Prerequisites

You need Python 3.8 or higher.
Install the required Python packages:

```bash
pip install numpy matplotlib Pillow
```

### Installation

1.  Clone the repository:

    ```bash
    git clone https://github.com/..../PD-FHC.git # Update with your actual repo
    cd PD-FHC
    ```

2.  **Prepare Cover Images**:
    The system requires a set of innocuous cover images for steganography.
    -   Create a directory named `C:/cover` (or another path, but you'll need to update the `cover_dir` variable in `main.py`).
    -   Populate this directory with `PNG` or `JPG` images of *identical dimensions*.
    -   You will need `2 * num_gates` images in total. For the example circuit (14 original gates, typically ~17 obfuscated gates), you'd need around `34` images. You can use any images you want, but ensure they are visually harmless.

### Running the Demonstration

To see the PD-FHC protocol in action with a sample circuit and manual configuration of secret locations:

```bash
python main.py
```

The script will:
1.  Display the original and obfuscated circuit structures.
2.  Prompt you to input the number of secret locations and their pixel coordinates (row, column, channel).
3.  Prompt you to provide binary input values (0 or 1) for the circuit's inputs (A, B, C) for *each* secret location.
4.  Simulate Alice sending data to Carol.
5.  Simulate Carol performing the homomorphic computation across the images. Carol's processing includes displaying images after each gate, demonstrating the pixel manipulation.
6.  Display the extracted results for each secret location (both real and decoys).
7.  Demonstrate a "coercion scenario" where Alice reveals a decoy computation, and Eve verifies its consistency.

**Interactive Plot Display**: During the demonstration, plots showing intermediate image states after each gate will be displayed. These plots are essential for visualizing Carol's computation.

### Running Benchmarks

The `main.py` script also includes a comprehensive benchmarking suite. When prompted:

```text
Do you want to run the comprehensive benchmarks? (y/N): y
```

This will run a series of benchmarks across various image sizes and numbers of secret locations. This process can be time-consuming. Upon completion:

1.  A summary table of performance metrics will be printed to the console.
2.  The full benchmark results will be saved to `pdfhc_benchmarks.json`.
3.  You will be asked if you want to display the generated plots interactively (`y/N`).
4.  Publication-quality plots will be generated and saved as PDF files in a `benchmark_plots/` directory.

## Benchmarking Results and Analysis

### Summary Table

An example of the summary table produced:

```
BENCHMARK PERFORMANCE SUMMARY
==============================================================================================================
Image Size   Locations  Gates (obf)   Total Time (ms) Per-Gate (ms)   Throughput (gates/s)  
--------------------------------------------------------------------------------------------------------------
64x64        2          17            4.87            0.286           3492                  
64x64        4          17            4.36            0.256           3901                  
64x64        8          17            4.41            0.259           3856                  
128x128      2          17            12.23           0.719           1391                  
... (and so on for other configurations)
==============================================================================================================
Highest throughput observed: 3901 gates/second
  Configuration: 64x64 pixels, 4 locations (17 obfuscated gates)
```

### Plots

The `benchmark_plots/` directory will contain several plots in PDF format, including:

-   `throughput_scaling.pdf`: Throughput (gates/second) vs. Image Size for different numbers of locations.
-   `multilocation_overhead.pdf`: Execution time vs. Image Size, illustrating overhead from multiple locations.
-   `per_gate_latency.pdf`: Average latency per gate (ms) vs. Image Size.
-   `scalability_analysis.pdf`: Normalized execution time vs. normalized pixel count, showing linear scaling.
-   `overhead_percentage.pdf`: Multi-location overhead as a percentage relative to a baseline (e.g., 2 locations).
-   `baseline_comparison.pdf`: Compares the vectorized PD-FHC implementation against a simulated naive approach.

These plots provide a visual understanding of PD-FHC's performance characteristics and scalability.

## Future Work

-   **GPU Acceleration**: Implement core `fredkin_gate_vectorized` operation using frameworks like `CUDA` or `OpenCL` for substantial speedups on larger image sizes and more complex circuits.
-   **More Gate Types**: Extend the framework to support additional universal reversible logic gates (e.g., Toffoli) for broader circuit compatibility.
-   **Advanced Obfuscation**: Explore more sophisticated circuit obfuscation techniques to further increase the adversary's difficulty in reverse-engineering.
-   **Dynamic Image Generation**: Implement on-the-fly generation of cryptographically random cover images to remove the dependency on pre-existing image datasets.
-   **Client-Side Tools**: Develop a more user-friendly client-side interface for Alice to define circuits, set up locations, and manage cover stories.
-   **Formal Verification**: Further explore formal methods for proving the security guarantees, particularly for the coercion game.
-   **Integration with Real FHE/TEE**: Investigate potential hybrid approaches where PD-FHC provides deniability on top of existing FHE or TEE solutions for multi-layered security.

## Contributing

We welcome contributions! If you have ideas for improvements, new features, or bug fixes, please open an issue or submit a pull request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.




---