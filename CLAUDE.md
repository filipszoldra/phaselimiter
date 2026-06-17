# CLAUDE.md — phaselimiter engine

This file guides Claude Code when working in this repository.

## What this repo is

`phaselimiter` is a C++ audio mastering engine. It is consumed by the `phaselimiter-gui` Go/GTK
frontend (sibling directory `../phaselimiter-gui`). The GUI builds a CLI invocation and runs
`phase_limiter.exe` as a child process; it does not call any library functions.

The upstream source is `github.com/ai-mastering/phaselimiter` (inactive). This fork lives at
`github.com/filipszoldra/phaselimiter`.

## Active branch

`engine-eq-transform` — current development branch: build fixes for modern toolchains (Intel
oneAPI IPP 2026, oneTBB 2021+) **plus** all the fork's new flags (per-band EQ limits, static EQ
correction, per-section wet/dry blend). This is the branch CI builds from and the one merged to
`master` for releases. (Earlier work lived on `engine-oneapi-ipp` / `engine-eq-band-levels`.)

## Build environment

**There is no local build toolchain.** No `cmake`, `cl`, `conda`, or `make` on PATH.
All builds happen in GitHub Actions CI. To trigger a build: commit + push to `engine-*` branch
and the `build-engine` workflow fires automatically.

### CI workflow: `.github/workflows/build-engine.yml`

- Runs on `windows-latest` (MSVC + conda-forge Miniconda)
- Installs **Intel oneAPI IPP 2026** via offline bootstrapper (cached at key `oneapi-ipp-2026.0.0.193`)
  - `IPPROOT = C:/Program Files (x86)/Intel/oneAPI/ipp/latest`
  - Lib names: `ippi.lib`, `ipps.lib`, `ippcore.lib`, `ippvm.lib` (NOT the old `ippimt.lib` etc.)
- Installs **oneTBB 2021+** from conda-forge (`tbb-devel`, no version pin)
  - DLL ships as `tbb12.dll` / `tbbmalloc12.dll` (NOT `tbb.dll`)
- Builds `--target phase_limiter --target audio_analyzer` (the GUI uses both: `audio_analyzer`
  powers track analysis / suggest / before-after; skips `audio_visualizer`, etc.)
- Uploads artifact `engine-bin` containing `bin/Release/` + all runtime DLLs

### Key dependencies and versions

| Dep | Source | Notes |
|-----|--------|-------|
| Intel IPP | oneAPI 2026 offline installer | Static libs, no redist DLL needed |
| TBB | conda-forge `tbb-devel` (oneTBB 2021+) | `tbb12.dll` in artifact |
| Boost 1.82 | conda-forge | boost_system/filesystem/serialization/math_tr1/iostreams |
| Armadillo 12.6 | conda-forge | |
| libpng 1.6 | conda-forge | |
| libsndfile | prebuilt `prebuilt/win64/libsndfile-1.2.2-win64/` | in-tree |

## Source changes vs upstream (engine-oneapi-ipp branch)

All changes were made to support oneTBB 2021+ and Intel oneAPI IPP. No algorithmic changes.

### `CMakeLists.txt` (Windows section)
- IPP lib names: `ippimt/ippsmt/ippcoremt/ippvmmt` → `ippi/ipps/ippcore/ippvm`
- Added `tbb.lib` and `tbbmalloc.lib` explicitly (oneTBB 2021+ dropped auto-link pragma)
- Added `link_directories(${IPPROOT}/lib)` and `include_directories(${IPPROOT}/include)`

### `src/phase_limiter/GradCalculator.h`
- Removed `#include "tbb/pipeline.h"` (header deleted in oneTBB 2021+; was unused anyway)
- `task_allocator_.construct(t, task)` → `::new (t) TaskType(task)` (C++20 removed allocator construct)
- `task_allocator_.destroy(tasks[j])` → `tasks[j]->~TaskType()` (same)

### `src/phase_limiter/main.cpp`
- Added `#include "tbb/global_control.h"` and `#include "tbb/info.h"`
- `tbb::task_scheduler_init` → `tbb::global_control` + `tbb::info::default_concurrency()`

## Fork-added flags (implemented, on `engine-eq-transform`)

All defined in `src/phase_limiter/main.cpp` (`DEFINE_*`) and consumed in
`src/phase_limiter/auto_mastering5.cpp`. Each is a no-op when empty / neutral, so zero regression
risk vs upstream. The GUI sends them only when non-default (`../phaselimiter-gui/mastering.go`).

| Flag | Type | Effect |
|---|---|---|
| `--mastering5_eq_band_levels` | CSV[9] | Per-band multiplier (1=neutral) on the optimizer's **wet-gain upper bound** (mid `8*i+1` & side `8*i+5`). Soft penalty in the cost function → proportional, not a hard clip. <1 restrains boosts/width, >1 permits more. |
| `--mastering5_eq_transform_levels` | CSV[9] | Per-band multiplier applied **after** optimization to the realized wet-gain, as half-delta `r = 1 + 0.5*(level-1)`. Deterministic per-band scaling, independent of the (soft) upper bound. |
| `--mastering5_eq_transform_symmetric` | bool | If true, transform also scales cuts (negative wet_gain); default false = boosts only (lowering a band never un-cuts it). |
| `--eq_analysis_target` | CSV[9] | Per-band relative dB deltas (`user_target - input`). Static EQ correction applied **after AutoMastering5, before pre-compression**: `gain_db = clamp((delta - normalized_spectral_change) * mastering_level, -6, +6)`. Scaled by intensity + clamped ±6 dB inside the engine. |
| `--mastering5_section_ranges` | CSV `start:end` | Time ranges (seconds) where the AM5 result is blended toward the **dry** (pre-AM) signal. Single-pass, with a **1 s raised-cosine ramp** at boundaries. |
| `--mastering5_section_intensity` | double | Wet/dry strength inside those ranges (0 = fully dry, 1 = full AM5). ~0.2–0.4 gentle-ifies quiet sections. |

The section blend captures `dry = *_wave` before band processing and, after reconstruction,
applies `(*_wave)[k] = dry[k] + w*((*_wave)[k]-dry[k])` per frame, where `w` ramps 1→intensity→1
across each range (raised-cosine, full-wet at boundaries → no click).

> Note: the `--mastering5_section_ranges` help string in `main.cpp` still says "100 ms" — the
> implemented ramp is **1 s** (`ramp_sec = 1.0f` in `auto_mastering5.cpp`). Cosmetic only.

**Do NOT** route the GUI's EQ correction through `--mastering5_mastering_reference_file`: that
flag switches the engine into a distance/reference-file mode and breaks the control surface. The
EQ correction is `--eq_analysis_target` only.

## Pipeline stages (quick reference)

Source: `src/phase_limiter/main.cpp` `MainFunc()`.

decode → band-cut → **AutoMastering5** (per-band M/S compressor via differential-evolution,
`auto_mastering5.cpp`; the fork adds per-band EQ/transform limits, the static EQ correction, and
the per-section wet/dry blend here) → **pre-compression** (`pre_compression.cpp`) →
gain-to-target-loudness → **phase limiter** (iterative FFT optimization, `GradCalculator.h`) →
true-peak ceiling → encode

Key flags (all passable without recompile):
- `--reference` — loudness target LUFS (engine default -6; the GUI sets -14; -12/-14 = less crunch)
- `--mastering5_mastering_level` — intensity 0–1 (engine default 0.5; GUI default 0.4)
- `--mastering5_eq_band_levels` / `--mastering5_eq_transform_levels` — per-band optimizer limits
- `--eq_analysis_target` — static per-band EQ correction (scaled by intensity, clamped ±6 dB)
- `--mastering5_section_ranges` / `--mastering5_section_intensity` — per-section wet/dry blend
- `--pre_compression_threshold` / `--pre_compression_mean_sec`
- `--ceiling` — true-peak ceiling (set ~-1.0 for headroom)

Runtime resource files (no recompile needed):
- `resource/mastering_reference.json` — reference profile for AutoMastering5
- `resource/progression_mapping.json`
- `resource/sound_quality2_cache/` — LOF index (independent of optimizer bounds changes)

## gh CLI usage (no persistent auth)

```bash
tok=$(printf "protocol=https\nhost=github.com\n\n" | git credential fill 2>/dev/null | sed -n 's/^password=//p')
GH_TOKEN="$tok" gh run list --repo filipszoldra/phaselimiter --workflow build-engine.yml
GH_TOKEN="$tok" gh run download <run-id> -n engine-bin -D ./engine-artifact
```
