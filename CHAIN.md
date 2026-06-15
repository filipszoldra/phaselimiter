# Processing chain — step by step

Every audio file goes through the following stages in `MainFunc()` (`src/phase_limiter/main.cpp`).
All stages operate at **44 100 Hz, stereo (interleaved float32)** in memory.

---

## Stage 0 — Decode (`FFMpeg::Execute`)

**What:** ffmpeg converts the input file to a raw float32 PCM buffer at 44 100 Hz, stereo.
Any format ffmpeg can read is accepted (WAV, FLAC, MP3, AIFF, …).

**Flags:** `--input`, `--ffmpeg`, `--disable_input_encode` (skip decode, load float WAV directly),
`--start_at` / `--end_at` (time crop, 0.5 s margin added automatically).

**Audible effect:** none — lossless reformat.

---

## Stage 1 — Low / high cut (`CutLowAndHighFreq`)

**What:** Linear-phase FIR low-cut and high-cut applied via `bakuage/fir_filter`.
Removes sub-bass rumble and ultrasonic content that would otherwise waste limiter headroom.

**Flags:** `--low_cut_freq` (default 20 Hz), `--high_cut_freq` (default 20 000 Hz).
Set either to 0 to disable.

**Audible effect:** negligible at defaults; narrowing either bound audibly thins bass or dulls highs.

---

## Stage 2 — (Optional) Enhancement / Freq expansion

Both disabled by default and guarded by `PHASELIMITER_ENABLE_FFTW` (not compiled into the
IPP build). Experimental / unfinished — do not use.

Flags: `--enhancement`, `--freq_expansion`, `--freq_expansion_ratio`.

---

## Stage 3 — Internal normalize (`Normalize`)

**What:** Scales the buffer so its L-∞ (peak) value is exactly 1.0.
Done purely to give the next stage a consistent input level.

**Audible effect:** none (gain is re-applied later to hit the loudness target).

---

## Stage 4 — AutoMastering5 (`AutoMastering5`)

**What:** The main tonal and stereo shaping stage.
Uses **Differential Evolution + Particle Swarm Optimization** (DE-PRMM / PSO-DV) to find
9-band mid/side wet-gain parameters that maximize a learned **SoundQuality2** score —
essentially matching a reference spectral/dynamics profile stored in
`phaselimiter/resource/mastering_reference.json`.

Each band has 8 parameters (mid wet gain, side wet gain, mid/side compressor ratio, knee, attack,
release, make-up, parallel blend — exact layout in `auto_mastering5.cpp`).

**Key flags:**
| Flag | Effect |
|------|--------|
| `--mastering5_mastering_level` 0–1 | How hard to reshape; lower = subtler |
| `--mastering_ms_matching_level` 0–1 | Stereo-field match strength |
| `--mastering5_eq_band_levels` CSV×9 | Per-band **ceiling**: multiplies the optimizer's upper bound (soft) |
| `--mastering5_eq_transform_levels` CSV×9 | Per-band **transform strength**: scales realized wet_gain post-optimization via `r = 1 + 0.5·(level−1)` |
| `--mastering5_eq_transform_symmetric` bool | If false (default): only scales boosts; if true: also scales cuts |
| `--mastering5_optimization_max_eval_count` | Max DE/PSO evaluations (default 40 000; fewer = faster, coarser) |

**Audible effect:** The biggest tonal and stereo change. At intensity 0.5 and target −14 LUFS
this typically adds 3–6 dB of perceived loudness, equalizes the spectrum toward the reference
profile, and adjusts the stereo width per band.

**Output flag:** `--output_after_automastering <path>` (see Stage 4b).

---

## Stage 4b — Save intermediate: after AutoMastering5

**What:** If `--output_after_automastering` is set, the buffer is encoded and saved to that
path immediately after Stage 4. Useful for A/B testing the AutoMastering5 contribution in
isolation (ceiling + normalisation will still be missing, so levels differ from the final).

---

## Stage 5 — Low / high cut + normalize (cleanup)

Same as Stages 1 + 3 applied again to remove any band-edge artifacts introduced by AutoMastering5.

---

## Stage 6 — Pre-compression (`PreCompress`)

**What:** A broadband RMS compressor that tames the loudest moments **before** the phase
limiter sees them. Reduces pumping and preserves more dynamics in the limiter stage.

Internally: sliding-window RMS → gain curve → soft-knee compression applied to the whole signal.

**Key flags:**
| Flag | Default | Effect |
|------|---------|--------|
| `--pre_compression` | true | Enable/disable |
| `--pre_compression_threshold` | +6 dB (above track loudness) | Higher → less compression, more dynamics |
| `--pre_compression_mean_sec` | 0.2 s | Window for RMS measurement; longer = smoother/less pumping |

**Audible effect:** Reduces dynamic peaks before limiting. More threshold offset → more
dynamics preserved; longer window → less pumping but slower transient response.

**Output flag:** `--output_after_pre_compression <path>` (built-in engine flag).

---

## Stage 7 — Low / high cut + normalize (cleanup again)

Same cleanup as Stage 5.

---

## Stage 8 — Loudness measurement + gain calculation

**What:** EBU R128 integrated loudness is measured over the entire buffer. A linear gain `g`
is calculated so that applying it would bring the track to `--reference` LUFS.

The gain also incorporates the ceiling offset: `r = 10^((gain − ceiling) / 20)` so that after
limiting the true peak does not exceed `--ceiling`.

**Key flags:** `--reference` (default −9, GUI default −14), `--reference_mode`
(loudness / youtube_loudness / rms / peak / zero).

**Audible effect:** none yet — the gain is calculated here but applied in Stage 9.

---

## Stage 9 — Apply gain

The buffer is multiplied by `r` from Stage 8. Any sample whose absolute value now exceeds
`1 + 0.5/65536` sets a `need_limiting` flag that controls whether Stage 10 runs.

---

## Stage 10 — Phase limiter (`PhaseLimitInplace`)

**What:** The engine's signature stage. An **iterative FFT-domain optimizer** minimizes
the total audible limiting error subject to the constraint that no sample exceeds ±1.0.
Unlike a time-domain brick-wall limiter (which just clips peaks), the phase limiter spreads
the excess energy across nearby time/frequency bins in a perceptually optimal way — reducing
clicks, crunch, and inter-modulation distortion.

Implementation: `src/phase_limiter/GradCalculator.h` + `GradCore.h`.
Algorithm: projected gradient descent with FISTA acceleration; inner/outer iteration loops.

**Key flags:**
| Flag | Default | Effect |
|------|---------|--------|
| `--max_iter1` | 100 | Outer iterations; more = fewer audible errors, much slower |
| `--max_iter2` | 400 | Inner iterations per outer step |
| `--limiter_internal_oversample` | 1 | In-algorithm oversampling; 2× or 4× reduces aliasing |
| `--limiter_external_oversample` | 1 | Resampling before/after; slow, high memory |
| `--erb_eval_func_weighting` | false | Perceptual ERB weighting of limiting error (preserve bass) |
| `--limiting_mode` | phase | `phase` (full algorithm) or `simple` (instant clip) |

**Audible effect:** Determines the quality ceiling. More iterations and higher oversampling
→ cleaner transients, fewer artifacts — at the cost of CPU time and RAM.

**Output flag:** `--output_after_phase_limiter <path>` (see Stage 10b).

---

## Stage 10b — Save intermediate: after phase limiter

**What:** If `--output_after_phase_limiter` is set, the buffer is saved here — after limiting
but before ceiling enforcement. Lets you hear the pure limiter output without the true-peak pass.

---

## Stage 11 — Ceiling enforcement

**What:** Ensures the output meets the true-peak ceiling (`--ceiling`, typically −1 dB or 0 dB).

Three modes (`--ceiling_mode`):
- `peak` — simple sample peak clamp
- `true_peak` (default) — 4× oversampled peak detection + final gain nudge
- `lowpass_true_peak` — adds a `--lowpass_true_peak_cut_freq` filter before peak detection

**Flags:** `--ceiling` (default 0 dB; set −1.0 for headroom), `--true_peak_oversample` (default 4).

---

## Stage 12 — Encode output (`FFMpeg::Execute`)

**What:** ffmpeg re-encodes the float32 buffer to the requested output format.
For lossy formats (AAC, MP3) a pre-encode pass runs **before** Stage 6 to remove ultrasonic
content that would otherwise cause post-encoding true-peak overshoots.

**Flags:** `--output`, `--output_format` (wav/mp3/aac), `--bit_depth` (16/24), `--sample_rate`.

---

## Quick-reference: intermediate output flags

| Flag | Stage saved after |
|------|------------------|
| `--output_after_automastering <path>` | Stage 4 (AutoMastering5) |
| `--output_after_pre_compression <path>` | Stage 6 (pre-compression) — built-in |
| `--output_after_phase_limiter <path>` | Stage 10 (phase limiter) |

---

## Typical loudness budget (−14 LUFS target, intensity 0.5)

| Stage | Typical integrated LUFS |
|-------|------------------------|
| Input | −18 … −22 |
| After AutoMastering5 | −16 … −18 (subtle reshape) |
| After pre-compression | −16 … −17 |
| After gain application | −14 (by definition) |
| Final output | −14 (true-peak ≤ −1 dBTP) |
