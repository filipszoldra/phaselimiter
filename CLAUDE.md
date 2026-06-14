# CLAUDE.md — phaselimiter engine

This file guides Claude Code when working in this repository.

## What this repo is

`phaselimiter` is a C++ audio mastering engine. It is consumed by the `phaselimiter-gui` Go/GTK
frontend (sibling directory `../phaselimiter-gui`). The GUI builds a CLI invocation and runs
`phase_limiter.exe` as a child process; it does not call any library functions.

The upstream source is `github.com/ai-mastering/phaselimiter` (inactive). This fork lives at
`github.com/filipszoldra/phaselimiter`.

## Active branch

`engine-oneapi-ipp` — contains all build fixes for modern toolchains (Intel oneAPI IPP 2026,
oneTBB 2021+). This is the branch CI builds from. Do not touch `master` without a good reason.

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
- Builds **only** `--target phase_limiter` (skips `audio_analyzer`, `audio_visualizer`, etc.)
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

## Planned changes (not yet committed)

### FAZA 1: `--mastering5_eq_band_levels` flag

Adds per-band multipliers on the AutoMastering5 optimizer's wet-gain upper bounds, allowing
the GUI to restrain aggressive frequency boosts/stereo widening on sparse-highs tracks without
switching out of quality mode (LOF model untouched, no mode switch, no model retraining).

**`src/phase_limiter/main.cpp`** — after line 69 (`DEFINE_string mastering5_mastering_reference_file`):
```cpp
DEFINE_string(mastering5_eq_band_levels, "",
    "Comma-separated per-band multipliers (>=0, 1=neutral) applied to optimizer "
    "wet-gain upper bound (mid & side). <1 restrains, >1 permits more. Empty=no-op.");
```

**`src/phase_limiter/auto_mastering5.cpp`** — after `upper_bounds *= scale;` (~line 348):
```cpp
// Add DECLARE_string(mastering5_eq_band_levels); near top (after existing DECLARE_ block)

if (!FLAGS_mastering5_eq_band_levels.empty()) {
    std::vector<double> levels;
    std::string s = FLAGS_mastering5_eq_band_levels;
    for (size_t p; (p = s.find(',')) != std::string::npos; s.erase(0, p+1))
        levels.push_back(std::stod(s.substr(0, p)));
    if (!s.empty()) levels.push_back(std::stod(s));
    if ((int)levels.size() == band_count) {
        for (int i = 0; i < band_count; i++) {
            upper_bounds(8*i+1) *= levels[i];  // mid wet gain
            upper_bounds(8*i+5) *= levels[i];  // side wet gain
        }
    }
}
```
Empty flag or wrong band count → no-op → zero regression risk.

## Pipeline stages (quick reference)

Source: `src/phase_limiter/main.cpp` `MainFunc()`.

decode → band-cut → **AutoMastering5** (per-band M/S compressor via differential-evolution,
`auto_mastering5.cpp`) → **pre-compression** (`pre_compression.cpp`) → gain-to-target-loudness →
**phase limiter** (iterative FFT optimization, `GradCalculator.h`) → true-peak ceiling → encode

Key flags (all passable without recompile):
- `--reference` — loudness target LUFS (default -9, very loud; -12/-14 = less crunch)
- `--mastering5_mastering_level` — intensity 0–1
- `--mastering5_eq_band_levels` — per-band multipliers (FAZA 1)
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
