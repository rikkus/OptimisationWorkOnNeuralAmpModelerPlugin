# NeuralAmpModelerPlugin — Apple Silicon optimization fork

This is a fork of `sdatkinson/NeuralAmpModelerPlugin` whose purpose is a
performance optimization of NAM's WaveNet inference on Apple Silicon (and other
ARMv8+ CPUs). The optimization work lives on the branch
**`optimise-for-apple-silicon`** in both this repo and the `NeuralAmpModelerCore`
submodule; the default `main` branch does **not** contain it.

## Where the work is

Almost all of it is in the `NeuralAmpModelerCore` submodule (a fork at
`rikkus/OptimisationWorkOnNeuralAmpModelerCore`; this repo's `.gitmodules`
points the submodule there):

- **`NeuralAmpModelerCore/NAM/wavenet/fused.{h,cpp}`** — a fused, register-tiled
  NEON WaveNet engine for AArch64. Standard-shape models (A1 standard/lite
  family, A2 standard, and similar: mono in, `bottleneck == channels`, channels
  a multiple of 4 ≤ 32, no gating/FiLM/grouping) are routed here instead of the
  generic Eigen path. Everything else falls through unchanged.
- **`NeuralAmpModelerCore/docs/fused-engine.md`** — **read this first.** The
  authoritative design doc: the profile that motivated it, the kernel design,
  the measured numbers, and the alternatives that were measured and rejected
  (Accelerate/AMX, fp16/bf16, BNNS/Metal/ANE, multithreading).
- **`NeuralAmpModelerCore/tools/test/test_fused.cpp`** — numerical parity tests
  (fused vs generic within 5e-5) across shapes, activations, and block sizes,
  plus detector negatives and a zero-allocation real-time-safety test. Wired
  into `run_tests`.
- Dispatch is in `NeuralAmpModelerCore/NAM/wavenet/model.cpp`
  (`wavenet::create_config`): order is slimmable → fused → a2_fast → generic.
- **`NeuralAmpModelerCore/benchmark_reports/`** — before/after Apple M2 reports
  (`..._apple_m2_baseline.txt` = generic; `..._192034.txt` = fused) and
  `run_benchmarks.sh`.

Plugin-side integration in this repo:

- `NeuralAmpModeler/config/NeuralAmpModeler-{mac,ios}.xcconfig` and
  `NeuralAmpModeler-win.props` define `NAM_ENABLE_FUSED` (and, for iOS which was
  previously missing them, `NAM_ENABLE_A2_FAST` too).
- The macOS and iOS Xcode projects add `wavenet/fused.cpp` (and, for iOS,
  `wavenet/a2_fast.cpp`) to the source lists.

The fused engine is gated by the `NAM_ENABLE_FUSED` compile definition
(CMake option `NAM_ENABLE_FUSED`, default ON); it is a no-op stub on non-ARM
builds, and the shape detector declines every model on those targets.

## Results (Apple M2, 48 kHz, buffer 64)

`wavenet_a1_standard` ~86 ms → ~39 ms per 2 s of audio (23× → 51× real-time,
**~2.2×**); ~2.7× at buffer 16; A2 standard ~2×. Fused-vs-generic output agrees
to −127 dB RMS on real rendered models. See `docs/fused-engine.md` for the full
table and methodology.

## Building and benchmarking the core (fast iteration path)

The core library builds standalone without the (heavy) iPlug2 submodule:

```sh
# From this repo root. Nested submodules must be present or CMake fails on
# AudioDSPTools/dsp/wav.cpp:
git -C NeuralAmpModelerCore submodule update --init --recursive

cd NeuralAmpModelerCore
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --target benchmodel benchmodel_bufsize bench_a2_fast run_tests render -j$(sysctl -n hw.ncpu)

./tools/run_tests                                    # correctness (incl. fused parity)
./tools/benchmodel example_models/wavenet_a1_standard.nam   # 2s @ 48k, buffer 64
../benchmark_reports/run_benchmarks.sh               # full timestamped report
```

`benchmodel` takes a `--seconds N` flag (added for longer profiling runs;
`sample <pid> 10` gives good flat profiles). To A/B against the generic path,
configure a second build dir with `-DNAM_ENABLE_FUSED=OFF` and diff `render`
output WAVs (they are float32 — parse the RIFF manually; Python's `wave` module
rejects format 3).

## Key design decisions (don't relitigate without new data)

- **Hand NEON, not Accelerate/AMX**: `cblas_sgemm` only ties the register-tiled
  NEON kernel at these tiny matrix sizes (~16×16 per tap); the AMX advantage
  needs much larger matrices. Not worth the dependency or unclear RT behavior.
- **`vdivq_f32` for the fast-tanh rational**, not reciprocal + Newton: the
  M-series FP divider is a separate unit, so division overlaps the FMAs and
  measured faster (0.26 vs 0.39 ns/float) — and it is exact.
- **fp16 gives no compute win** on Apple cores (`FMLAL` is the same 4
  MACs/instruction as fp32 FMA; we are compute-bound). Full fp16 accumulation
  is an audio-quality risk over the long conv sums. **SME (M4+)** is the real
  next lever but needs M4 hardware to develop/validate.
- Activations use NEON kernels only for known implementations (fast-tanh, ReLU,
  LeakyReLU, Hardtanh, Softsign); anything else calls the exact same
  `Activation` object the generic path would, so semantics (including
  `enable_fast_tanh()` and LUTs) never diverge.

## Fork / branch layout

- This repo: `origin` = `rikkus/OptimisationWorkOnNeuralAmpModelerPlugin`,
  upstream = `sdatkinson/NeuralAmpModelerPlugin`. Work on
  `optimise-for-apple-silicon`.
- Core submodule fork: `rikkus/OptimisationWorkOnNeuralAmpModelerCore`, same
  branch name. The submodule commit referenced by this branch
  (`09d46b0`, the fused engine) lives on that fork's `optimise-for-apple-silicon`.
- These forks keep the `OptimisationWorkOn…` name because GitHub allows only one
  fork of a given upstream per account; renaming (with redirects) is the way to
  get a shorter name if ever wanted.
