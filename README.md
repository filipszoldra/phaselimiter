# phaselimiter (filipszoldra fork)

A high-quality audio limiter and automated mastering engine written in C++.
The same algorithm used on [bakuage.com](https://bakuage.com) / [aimastering.com](https://aimastering.com).

Originally: 音圧爆上げくんで使われているリミッターと自動マスタリング

This is a fork of [ai-mastering/phaselimiter](https://github.com/ai-mastering/phaselimiter)
maintained by [@filipszoldra](https://github.com/filipszoldra). The upstream is inactive.

## What this fork changes vs. upstream

### Build system — Intel oneAPI IPP 2026

The upstream targets a discontinued Intel conda channel (HTTP 403 since ~2024) and the
old TBB 2019 API. This fork fixes the build:

| File | Change |
|---|---|
| `.github/workflows/build-engine.yml` | New lean workflow: Intel oneAPI IPP 2026 (offline installer, cached), conda-forge oneTBB 2021+/Boost/Armadillo/libsndfile, uploads `engine-bin` artifact with exe + all DLLs |
| `CMakeLists.txt` | Updated Windows IPP lib names (`ippimt.lib` → `ippi.lib` etc.), explicit `tbb.lib`/`tbbmalloc.lib`, `IPPROOT` include/link paths |
| `GradCalculator.h` | Removed `#include "tbb/pipeline.h"` (deleted in oneTBB 2021+); replaced deprecated `cache_aligned_allocator::construct()/destroy()` with placement new / explicit destructor |
| `src/phase_limiter/main.cpp` | Replaced `tbb::task_scheduler_init` with `tbb::global_control` + `tbb::info::default_concurrency()` (oneTBB 2021+ API) |

The upstream `build` and `build-win` workflows are silenced on `master` — only the new
`build-engine` workflow runs, triggered on `engine-*` branches.

### New engine flags (branch `engine-eq-transform`)

All added in `src/phase_limiter/{main.cpp,auto_mastering5.cpp}`. Each is a no-op when empty /
neutral, so there is zero regression risk vs upstream. The LOF quality model is unchanged and
there is no mode switch.

**Per-band optimizer limits** — restrain how aggressively AutoMastering5 reshapes each band:

```
--mastering5_eq_band_levels       1,1,1,1,1,0.6,0.5,0.4,0.4   # 9 multipliers, 1 = neutral
--mastering5_eq_transform_levels  1,1,1,1,1,1,1,1,1           # 9 multipliers, 1 = neutral
--mastering5_eq_transform_symmetric  false
```

- `--mastering5_eq_band_levels` scales the optimizer's per-band **wet-gain upper bound** (mid
  `8*i+1` & side `8*i+5`). The bound is soft (a `bound_error * 1e4` penalty in the cost function,
  not a hard clip), so the effect is proportional. <1 restrains boosts/width, >1 permits more.
- `--mastering5_eq_transform_levels` scales the **realized** wet-gain *after* optimization, as a
  half-delta `r = 1 + 0.5*(level-1)` — a deterministic per-band transform strength, independent of
  the soft upper bound above.
- `--mastering5_eq_transform_symmetric` (default false): when true the transform also scales cuts;
  when false it scales only boosts, so lowering a band never un-cuts it.

**Audible effect:** limits kick deformation and hihat over-widening on tracks with sparse
high-frequency content — the AI will not try to "fill in" what isn't there.

**Static EQ correction** — a user-drawn per-band tonal target:

```
--eq_analysis_target  0,0,-1.5,0,0,2,0,0,3   # 9 relative dB deltas (target - input), 0 = no change
```

Applied as a static per-band gain **after AutoMastering5, before pre-compression**:
`gain_db = clamp((delta[i] - normalized_spectral_change[i]) * mastering5_mastering_level, -6, +6)`.
The amount is scaled by intensity and clamped to ±6 dB inside the engine.

**Per-section wet/dry blend** — gentle-ify quiet sections in a single pass:

```
--mastering5_section_ranges     0:21.5,106.5:124   # CSV of 'start:end' seconds
--mastering5_section_intensity  0.25               # 0 = fully dry, 1 = full AutoMastering5
```

Inside each range the AutoMastering5 result is blended toward the **dry** (pre-AM) signal by
`section_intensity`, with a **1 s raised-cosine ramp** at each boundary (full-wet at the boundary
itself → no click). Replaces the old GUI-side re-render + splice approach.

## Prebuilt binaries

This fork's CI artifacts: see the **Actions** tab → `build-engine` workflow → latest successful run.

Upstream releases (IPP 2019 build): https://github.com/ai-mastering/phaselimiter/releases

## Building

**Local build is not supported** — the engine requires the proprietary Intel IPP library.
All builds run in GitHub Actions CI.

### CI: `build-engine` workflow

Triggered on push to any `engine-*` branch (or manually via workflow dispatch).

Steps:
1. Checkout with all submodules
2. Install Intel oneAPI IPP 2026 (cached between runs, ~600 MB)
3. Install oneTBB 2021+, Boost 1.82, Armadillo 12.6, libsndfile via conda-forge
4. Build `phase_limiter` + `audio_analyzer` targets
5. Collect exe + all runtime DLLs (IPP, TBB, Boost, sndfile, libbz2, zlib, zstd, liblzma)
6. Upload as artifact `engine-bin`

### Runtime dependencies (included in artifact)

`engine-bin/bin/` contains:
- `phase_limiter.exe`
- `audio_analyzer.exe` (used by the GUI for track analysis / suggest / before-after)
- `sndfile.dll`
- `tbb12.dll`, `tbbmalloc.dll` (oneTBB)
- `boost_system.dll`, `boost_filesystem.dll`, `boost_serialization.dll`, `boost_math_tr1.dll`, `boost_iostreams.dll`
- `libbz2.dll`, `zlib.dll`, `zstd.dll`, `liblzma.dll`
- Intel IPP DLLs (`ippcore.dll`, `ippi.dll`, `ipps.dll`, `ippvm.dll` + ISA variants)

Also required at runtime (not in artifact):
- `ffmpeg.exe` on PATH or beside the executable
- `resource/` directory (`sound_quality2_cache/`, `mastering_reference.json`, etc.) — taken from an upstream release, not regenerated by this fork

## Key flags

```
phase_limiter \
  --mastering true --mastering_mode mastering5 \
  --reference -9 \
  --mastering5_mastering_level 0.8 \
  --mastering_ms_matching_level 1.0 \
  --ceiling -1.0 \
  --limiter_internal_oversample 2 \
  --pre_compression true \
  --pre_compression_threshold 6.0 \
  --pre_compression_mean_sec 0.2 \
  --mastering5_eq_band_levels 1,1,1,1,1,0.6,0.5,0.4,0.4 \
  --input input.wav --output output.wav
```

| Flag | Effect |
|---|---|
| `--reference` | Loudness target LUFS (default -9; use -12/-14 for less limiting) |
| `--mastering5_mastering_level` | Intensity 0–1 (how hard AutoMastering5 reshapes the tone) |
| `--mastering_ms_matching_level` | Stereo-field match strength 0–1 (0 = ignore stereo) |
| `--ceiling` | True-peak ceiling dB (use -1.0 for headroom before lossy encode) |
| `--limiter_internal_oversample` | Oversampling factor (1/2/4 — higher reduces aliasing crunch) |
| `--max_iter1` | Limiter FISTA iterations (default 100; 200 = cleaner, slower) |
| `--pre_compression` | Pre-compressor before limiter (true/false) |
| `--pre_compression_threshold` | Pre-comp threshold offset dB above reference |
| `--pre_compression_mean_sec` | Pre-comp averaging window seconds (longer = less pumping) |
| `--mastering5_eq_band_levels` | Per-band optimizer upper-bound multiplier CSV (9 values, 1.0 = neutral) |
| `--mastering5_eq_transform_levels` | Per-band realized wet-gain multiplier CSV after optimization (9 values) |
| `--mastering5_eq_transform_symmetric` | If true, transform scales cuts too (default false = boosts only) |
| `--eq_analysis_target` | Static per-band EQ correction CSV (9 dB deltas; scaled by intensity, clamped ±6 dB) |
| `--mastering5_section_ranges` | CSV of `start:end` second ranges for the wet/dry section blend |
| `--mastering5_section_intensity` | Wet/dry strength inside section ranges (0 = dry, 1 = full AM5) |
| `--erb_eval_func_weighting` | Perceptual bass weighting in limiter (true/false) |

Full flag list: `src/phase_limiter/main.cpp` (`DEFINE_*` macros).

## License

MIT
