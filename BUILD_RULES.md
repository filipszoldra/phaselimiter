# Build rules — known issues and fixes

This file documents every non-obvious build problem encountered with this fork
(`filipszoldra/phaselimiter`). Read it before touching `.github/workflows/build-engine.yml`
or `CMakeLists.txt`. All issues listed here have been hit at least once in CI.

---

## 1. CMake policy — `CMAKE_POLICY_VERSION_MINIMUM=3.5` is required

**Problem:** `deps/gflags/CMakeLists.txt:73` calls `cmake_minimum_required` with a version
below 3.5. Newer CMake (shipped with VS 2026 on `windows-latest`) removed backwards
compatibility for policies below 3.5 and aborts with:
```
CMake Error at deps/gflags/CMakeLists.txt:73 (cmake_minimum_required):
  Compatibility with CMake < 3.5 has been removed from CMake.
```
**Fix:** Pass `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` on the configure command line.
Already in `build-engine.yml`. **Do not remove this flag.**

---

## 2. Intel IPP — use oneAPI 2026 offline installer, NOT conda channel

**Problem:** The Intel conda channel (`intel` channel) that shipped `ipp-static` and
`ipp-include` 2019 returns HTTP 403. The package is gone from the public channel.

**Fix:** Download the oneAPI offline bootstrapper directly from Intel's CDN and install
only `intel.oneapi.win.ipp.devel`. Cached under key `oneapi-ipp-2026.0.0.193`.

- Cache path: `C:\Program Files (x86)\Intel\oneAPI\ipp`
- `IPPROOT` env: `C:/Program Files (x86)/Intel/oneAPI/ipp/latest`
- Linked libs: `ippi.lib ipps.lib ippcore.lib ippvm.lib`
- `CMakeLists.txt` already updated to find 2026 layout.

**Do not revert** to the `intel` conda channel or `ipp-static`/`ipp-include` packages.

---

## 3. TBB — use conda-forge, NOT Intel conda channel

**Problem:** `tbb-devel` from Intel conda channel also HTTP 403.

**Fix:** `conda install -y -c conda-forge tbb-devel`. Already in workflow.
`CMakeLists.txt` links `tbb.lib tbbmalloc.lib`; `GradCalculator.h` uses the new
task-group API (removed `tbb/pipeline.h`); `main.cpp` uses `tbb::global_control`
instead of the removed `task_scheduler_init`.

---

## 4. `liblzma.dll` must be copied to the artifact

**Problem:** `boost_iostreams` from conda-forge ≥ 1.82 links against `liblzma.dll`
(LZMA support). The DLL ships in conda env but is not in PATH on the end-user machine,
so the exe fails at runtime if omitted.

**Fix:** `build-engine.yml` "collect binaries" step explicitly copies `liblzma.dll`
from `$CONDA_DLL_ROOT`. **Do not remove it from the copy list.**

---

## 5. IPP DLL paths vary by oneAPI release — use `find`, not hardcoded paths

**Problem:** The subdirectory under `C:\Program Files (x86)\Intel\oneAPI\ipp\` that
contains `ipp*.dll` changes between oneAPI versions (e.g. `2026.0.0/redist/intel64/`
vs older layouts).

**Fix:** The "collect binaries" step uses:
```bash
find "C:/Program Files (x86)/Intel/oneAPI/ipp" -name "ipp*.dll" -exec cp {} /tmp/results/bin/ \;
```
followed by an existence check. Do not hardcode the version subdirectory.

---

## 6. `PHASELIMITER_ENABLE_FFTW` is NOT defined — enhancement/freq_expander are dead code

`enhancement.cpp` and `freq_expander.cpp` are guarded by `#ifdef PHASELIMITER_ENABLE_FFTW`.
This fork's build does **not** define that macro (IPP provides FFT). Both `Enhance()` and
`FreqExpand()` compile to empty stubs. The `--enhancement` / `--freq_expansion` flags exist
but are no-ops at runtime. Do not expect them to produce output.

---

## 7. `--mastering5_eq_band_levels` is a soft bound only — not a hard EQ

The flag multiplies `upper_bounds(8i+1)` and `upper_bounds(8i+5)` in `auto_mastering5.cpp`.
The hard-bound path is disabled (`#if 0` around `vals_bound`). The bound is also only used
as the DE/PSO initial sampling range (`de_initial_lb/ub`, `pso_initial_lb/ub`) plus a soft
penalty (`bound_error * 1e4`). At low intensity the optimizer rarely hits the ceiling, so
the effect is emergent, not deterministic.

Use `--mastering5_eq_transform_levels` (added in branch `engine-eq-transform`) for a
**deterministic** post-optimization wet_gain scaling instead.

---

## 8. `wet_gain` is signed — boost-only is the sane default for transform scaling

`wet_gain` lower bound is `−1` (the optimizer can cut bands). `ToWetGain(x) = 10*x` maps
the parameter to dB. A symmetric multiplier `< 1` applied to a **negative** wet_gain makes
the cut smaller (band gets louder) — the opposite of user intent. Hence:

- `--mastering5_eq_transform_symmetric false` (default) = scale only positive wet_gain (boosts).
- `--mastering5_eq_transform_symmetric true` = scale all wet_gain including cuts.

Never change the default to `true` without user confirmation.

---

## 9. 8.3 paths required when the GUI launches the engine

`phase_limiter.exe` calls `ffmpeg` internally **without quoting paths**. Any path with
spaces (e.g. `Muzyka 2026`) causes silent failures. The GUI's `mastering.go` converts
all paths via `toShortPath()` before building the argument list. Keep that call in place.

---

## 10. `windows-latest` runner tracks GitHub's latest image — expect silent upgrades

The `runs-on: windows-latest` runner has been updated to VS 2026 / MSVC 19.51 at least
once mid-project (triggered issue #1 above). If a clean build that previously passed
suddenly fails at CMake configure, check whether the runner image bumped CMake or MSVC.
The fix is usually a CMake policy flag or a minor API change in deps.

---

## Branch / artifact layout (as of 2026-06-15)

| Branch | Purpose | Last good run |
|--------|---------|--------------|
| `engine-oneapi-ipp` | IPP 2026 + TBB migration base | 27498711694 ✅ |
| `engine-eq-band-levels` | `--mastering5_eq_band_levels` flag | 27498716620 ✅ |
| `engine-eq-transform` | `--mastering5_eq_transform_levels` + `_symmetric` | in progress |

Artifact `engine-bin` always contains:
- `bin/phase_limiter.exe`
- `bin/phase_limiter-console.exe` (if built)
- All runtime DLLs: `sndfile.dll`, `boost_*.dll`, `tbb*.dll`, `liblzma.dll`, `zlib.dll`,
  `zstd.dll`, `libbz2.dll`, `ipp*.dll`

Resource directory (`phaselimiter/resource/`) is **not** in the artifact — reuse the one
from an earlier artifact (e.g. `build-results5/phaselimiter/resource/`).
