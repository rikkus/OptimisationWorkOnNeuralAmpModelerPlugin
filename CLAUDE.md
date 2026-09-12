# NeuralAmpModelerPlugin — ARM optimization fork

This is a fork of `sdatkinson/NeuralAmpModelerPlugin` whose purpose is getting
faster WaveNet inference into the plugin on ARM: Apple Silicon, AArch64
generally, and 32-bit ARMv7. The default `main` branch does **not** contain the
work; it lives on the branches below.

## Where the work is

The kernels are in NeuralAmpModelerCore, proposed upstream as
[sdatkinson/NeuralAmpModelerCore#313](https://github.com/sdatkinson/NeuralAmpModelerCore/pull/313)
from the Core fork (`rikkus/OptimisationWorkOnNeuralAmpModelerCore`, branch
`apple-silicon-a2-planar`). This repo only wires them into the plugin builds,
proposed as
[sdatkinson/NeuralAmpModelerPlugin#679](https://github.com/sdatkinson/NeuralAmpModelerPlugin/pull/679).

In the Core submodule:

- **`NAM/wavenet/a2_planar.{h,cpp}`** — planar NEON kernels for the A2 fast
  path (A2 nano, 3 channels; A2 standard, 8 channels). **Read the header comment
  in `a2_planar.h` first**: it is the authoritative design note — the gate, the
  per-architecture tile widths and conv-loop shapes, the measured numbers, and
  the ARMv7 caveats.
- Selection happens in `A2FastConfig::create` (`NAM/wavenet/a2_fast.cpp`), which
  prefers the planar model where one exists and otherwise returns the reference
  `A2FastModel` (`create_a2_fast_reference_model`).
- **`tools/test/test_a2_planar.cpp`** — `memcmp` bit-identity against `a2_fast`
  at 14 block sizes, plus a check that the dispatcher really routes to planar.
  Wired into `run_tests`.
- **`tools/bench_a2_planar.cpp`** — renders a whole signal through both engines,
  compares bit for bit, and reports speed only if they matched.

Plugin-side integration in this repo (PR #679):

- The macOS, iOS and Windows projects register `wavenet/a2_planar.{cpp,h}`
  alongside `wavenet/a2_fast.{cpp,h}`. There is **no new build flag**:
  `a2_planar.h` gates itself on `NAM_ENABLE_A2_FAST` plus the target being
  AArch64, or 32-bit ARM with NEON and FMA. Everywhere else the translation unit
  compiles to no symbols (the x86_64 slice of a universal macOS binary has none).
- iOS additionally gets `NAM_ENABLE_A2_FAST` defined and `a2_fast.{cpp,h}`
  registered, which it didn't have before.
- All three platforms also register Core files the old submodule pin predated
  (`linear`, `nam_file`, `sequential`, and some headers); without them the link
  fails on `nam::validate_nam_file`.

## The submodule pin

`.gitmodules` points `NeuralAmpModelerCore` at **upstream**
(`sdatkinson/NeuralAmpModelerCore`) — a PR here must not change where the
submodule comes from. But until Core PR #313 merges, the pinned commit exists
only on the Core fork, so a plain `git submodule update --init` will fail to
fetch it. Fetch it by SHA from the fork instead:

```sh
SHA=$(git ls-tree HEAD NeuralAmpModelerCore | awk '{print $3}')
git -C NeuralAmpModelerCore fetch https://github.com/rikkus/OptimisationWorkOnNeuralAmpModelerCore.git "$SHA"
git -C NeuralAmpModelerCore checkout "$SHA"
git -C NeuralAmpModelerCore submodule update --init --recursive
```

Once #313 merges, repoint nothing — just bump the pin to an upstream commit.

## Results

Bit-identical to `a2_fast` in every case (not "within a tolerance"). At
64-frame blocks, A2 standard / A2 nano, against `a2_fast`:

| Part | A2 standard | A2 nano |
|---|---:|---:|
| Apple M2 | 2.46× | 2.02× |
| Cortex-A76 (Raspberry Pi 500) | 2.15× | 3.08× |
| Cortex-A17 (RK3288, 1416 MHz) | 1.35× | 1.55× |

PR #313's description has the methodology and more figures. Results are tracked
over time in Bencher from the benchmark harness at `rikkus/NAMBench`.

## Building and testing the core (fast iteration path)

The core library builds standalone without the (heavy) iPlug2 submodule:

```sh
# Nested submodules must be present or CMake fails on AudioDSPTools/dsp/wav.cpp:
git -C NeuralAmpModelerCore submodule update --init --recursive

cd NeuralAmpModelerCore
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target run_tests bench_a2_planar -j"$(sysctl -n hw.ncpu)"

./build/tools/run_tests
./build/tools/bench_a2_planar --submodel widest    example_models/A2.nam
./build/tools/bench_a2_planar --submodel narrowest example_models/A2.nam
```

`-DNAM_DISABLE_A2_PLANAR` turns the kernels off, which makes an A/B against the
reference a one-flag change. Build at `-O3`, never `-ffast-math`/`-Ofast`: it
lets the compiler contract across statements, which is exactly the freedom the
bit-identity claim depends on not being taken — in either engine.

## Building and testing the plugin

Follow `CONTRIBUTING.md`'s testing checklist. The CI recipe
(`.github/workflows/build-native.yml`) fetches the SDKs first:

```sh
(cd iPlug2/Dependencies/IPlug && ./download-iplug-sdks.sh)
(cd iPlug2/Dependencies && ./download-prebuilt-libs.sh)
```

Then build individual targets rather than `All` (the AAX SDK isn't public, so
`All` fails without it):

```sh
cd NeuralAmpModeler
xcodebuild -project ./projects/NeuralAmpModeler-macOS.xcodeproj \
  -xcconfig ./config/NeuralAmpModeler-mac.xcconfig DEMO_VERSION=0 \
  -target VST3 -configuration Release        # likewise APP, AU
```

Builds install to `~/Applications` and `~/Library/Audio/Plug-Ins/`,
**overwriting any NAM already installed there**.

- The project targets macOS 10.15. Current beta Xcode only accepts 12.0 and
  up, so locally raise `MACOSX_DEPLOYMENT_TARGET` in `common-mac.xcconfig` and
  the macOS project — and revert it before committing (the build also rewrites
  `LSMinimumSystemVersion` in `NeuralAmpModeler/resources/*-Info.plist`).
- **VST3:** Steinberg's command-line `validator` (build it from the full
  `steinbergmedia/vst3sdk`; iPlug2's copy of the SDK omits the hosting samples)
  runs the same suite as the VST3PluginTestHost's unit-test tab:
  `validator -e ~/Library/Audio/Plug-Ins/VST3/NeuralAmpModeler.vst3`.
- **AU:** `auval -v aufx 1YEo SDAa`.
- The Slim knob is hidden until a slimmable model is loaded; then an icon
  appears right of the model box. Slim < 0.5 selects A2 nano, ≥ 0.5 A2 standard
  — test both, since they are separate kernels.

## Known issues

- **(Fixed, but worth knowing.)** `test_a2_planar` used to fail at
  `channels=3, block=1` on GCC targets (a Cortex-A76, a Cortex-A17) while
  passing on an M2. Nothing was wrong with the kernels: upstream compiles the
  whole `run_tests` target at `-O0` for its allocation tracking, and GCC only
  contracts `a * b + c` into an FMA in its optimisers — so the *reference*
  `A2FastModel<3>` stopped being the code the bit-identity claim is about.
  Clang contracts during codegen, which is why Apple Silicon hid it. Core PR
  #313 now builds the two A2 kernels as an `-O3` object library for that target
  only. If a parity test ever fails on one machine and not another, suspect the
  optimisation level of the reference before the kernel.
- ARMv7 bit-identity rests on the host having FPSCR.FZ set (AArch32 NEON is
  always flush-to-zero; VFP scalar honours the bit). See `a2_planar.h`.
- The old-style (directory) model item in `CONTRIBUTING.md` can't be tested:
  the model picker only accepts `.nam`, and Core's `get_dsp_legacy` is declared
  but not defined (upstream `main` too).
- Code style: use the clang-format version Core's CI names (19). Newer versions
  reformat untouched upstream files; never commit a `format.bash` run
  wholesale.

## Fork / branch layout

- This repo: `origin` = `rikkus/OptimisationWorkOnNeuralAmpModelerPlugin`,
  upstream = `sdatkinson/NeuralAmpModelerPlugin`.
  - `fused-optimisation` — **the head of PR #679**. The name is historical: it
    once carried an earlier engine ("fused") that has since been retired in
    favour of the planar kernels. Left unrenamed so the open PR isn't disturbed.
  - `ir-optimisation` — parked IR work (see below).
- Core fork: `rikkus/OptimisationWorkOnNeuralAmpModelerCore`.
  - `apple-silicon-a2-planar` — the head of Core PR #313, ARMv7 work included.
  - `ir-optimisation` — test/bench harness for the IR work, built on the
    retired engine's commit.
- AudioDSPTools fork: `rikkus/AudioDSPTools`, branch `ir-optimisation` —
  zero-latency partitioned-FFT convolution for long cab IRs. Never proposed
  upstream, and upstream has nothing equivalent; the Plugin and Core
  `ir-optimisation` branches only wire it up.
- These forks keep the `OptimisationWorkOn…` name because GitHub allows only one
  fork of a given upstream per account; renaming (with redirects) is the way to
  get a shorter name if ever wanted.
