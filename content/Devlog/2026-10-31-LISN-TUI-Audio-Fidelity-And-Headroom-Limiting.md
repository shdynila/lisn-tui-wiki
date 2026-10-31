---
title: "Devlog: Eliminating Audio Distortion, Vocal Overdrive, and Buffer Underruns"
date: 2026-10-31
tags: ["lisn-tui", "devlog", "dsp", "audio", "headroom", "limiter", "termux", "mdx-net"]
---

# Devlog: Eliminating Audio Distortion, Vocal Overdrive, and Buffer Underruns

## Context & Motivation
Following the launch of the mobile Termux distribution, listening feedback revealed an urgent audio quality issue:
> *"the music quality isn't that good, it seems like the volume has been increased alot, and it sounds more broken"*

Upon running frequency analysis and examining the signal chain, three distinct distortion sources were discovered:
1. **Artificial Vocal Overdrive**: `pkg/engine/mdx.go` was applying up to $3.0\times$ ($+9.5\text{ dB}$) gain boost to isolated vocal tracks. This drowned out the rhythm section, amplified subtle neural isolation bleed, and destroyed mix balance.
2. **Aggressive Non-Linear Saturation**: The real-time playback thread and mixdown renderer applied `math.Tanh` unconditionally across all samples. When stems were summed, peaks between $2.0$ and $3.5$ were crushed against the ceiling, turning smooth musical transients into square-wave fuzz.
3. **Termux Buffer Starvation**: A $50\text{ms}$ buffer size was triggering PulseAudio underruns over the Android localhost TCP socket, resulting in clicks and stuttering.

---

## What We Changed

### 1. Acoustic Balance Restored ($Vocals + Instrumental = Original$)
- Removed the artificial $3.0\times$ vocal gain formula in `pkg/engine/mdx.go`.
- Vocals and instrumental now preserve their exact natural mix proportions directly from the neural separation model.

### 2. Transparent Soft-Knee Limiter (`SoftClip`)
- Created `pkg/audio/limiter.go` featuring a piecewise $C^1$-continuous limiter.
- For all samples below $0.95$ ($>99.9\%$ of normal playback), the audio passes through with **0% harmonic distortion (pure linear unity gain)**.
- Peaks exceeding $0.95$ are smoothly compressed asymptotically to $\pm 1.0$ using a soft knee, preventing harsh digital clipping or integer wrapping.

### 3. Sub-Stem Limiter Protection
- Updated ffmpeg filter chains in `pkg/engine/onnx.go` to include `alimiter=limit=0.95` on all boosted frequency bands, preventing 16-bit integer WAV clipping.

### 4. Buffer Latency Bump to 100ms
- Increased `speaker.Init` buffer in `pkg/audio/engine.go` from $50\text{ms}$ to $100\text{ms}$ (`sampleRate.N(time.Second / 10)`).
- Provides a stable buffer margin that eliminates PulseAudio dropouts on Android phones and PRoot environments without affecting UI responsiveness.

### 5. Cache Engine Bump (`v3`)
- Updated `ComputeCacheKey` with `CacheEngineVersion = "v3"` to ensure previously separated, distorted stems are superseded by clean, uncompressed audio.

---

## Verification
- Added `pkg/audio/limiter_test.go` verifying 100% linear pass-through below $0.95$, smooth limiting above $0.95$, and odd symmetry.
- All Go unit tests passed cleanly (`go test ./...`).
- Automated multi-architecture release workflow triggered for `v0.2.1`.
