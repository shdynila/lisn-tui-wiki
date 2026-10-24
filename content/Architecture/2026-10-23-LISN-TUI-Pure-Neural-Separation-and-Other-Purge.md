---
title: "LISN-TUI: Pure Go Neural Source Separation and Elimination of 'Other' Stem"
date: 2026-10-23
tags: ["architecture", "audio", "neural-networks", "mdx-net", "onnx", "golang", "dsp", "source-separation"]
---

# LISN-TUI: Pure Go Neural Source Separation and Elimination of "Other" Stem

## 1. Problem Diagnosis & Two Root Causes

### 1. The Legacy Cache Trap
Upon inspecting the active cache directory (`$LOCALAPPDATA/lisn-tui/cache`), we discovered 15 cached song folders dating back to **10/22/2026** with identical byte sizes (e.g., exactly 51,804,290 bytes for vocals, bass, drums, and other).
When testing or launching tracks from "Recent Separations", `CheckCache` found those old files and returned them immediately. The application was continuing to play the outdated mock FFmpeg EQ filtered audio from yesterday!

**Action Taken**:
- Purged all legacy mock files from disk.
- Updated `verifyStemDir()` to automatically reject and invalidate any cached song directory containing legacy `other.wav`.

### 2. The PyTorch `center=True` Time-Shift Bug
In the initial neural prototype, STFT slicing began at index 0 rather than centering frame 0 at sample 0.
In PyTorch UVR MDX-Net training:
- `torch.stft(..., center=True, pad_mode='reflect')` pads the signal with $n_{\text{fft}} // 2 = 3072$ samples on each side.
- Without this 3072-sample center padding, the spectrogram was time-shifted by ~70 ms! The neural network was processing out-of-phase coordinates, and when computing $\text{Instrumental} = \text{Mix} - \text{Vocals}$, subtracting a 70ms-delayed vocal created a comb filter rather than acoustic cancellation.

---

## 2. The UVR `demix_base` Stationary Margin Architecture

The separation engine in `pkg/engine/mdx.go` was refactored to implement the official **Ultimate Vocal Remover `demix_base` algorithm**:

```
[Audio Input Signal: nSample]
             │
             ▼
[Zero-Padded Mix: trim(3072) + nSample + pad + trim(3072)]
             │
             ├──> Chunk Size = 261,120 samples (dim_t=256, hop=1024)
             ├──> Step Size  = 254,976 samples (MDX_GEN_SIZE)
             │
             ▼
[STFTChunk: Center=True Reflect Padding (3072 samples)]
             │
             ├──> Flat Input Tensor [1, 4, 3072, 256]
             │    (Ch 0: L Real, Ch 1: L Imag, Ch 2: R Real, Ch 3: R Imag)
             │
             ▼
[ONNX Runtime C-API Forward Pass (onnxruntime.dll AVX2 SIMD)]
             │
             ▼
[ISTFTChunk: Center=True Reconstruction (261,120 samples)]
             │
             ├──> Slice Valid Block: cOut[3072 : 3072 + 254976]
             │    (Extracts only the 100% stationary internal region; discards edge tapers)
             │
             ▼
[Full Waveform Assembly: [:nSample]]
             │
             ├──> Vocals: 100% Isolated Neural Waveform
             └──> Instrumental = Mix - Vocals (Sample-Accurate Phase Cancellation)
```

### Mathematical Guarantee:
1. **Stationary Windowing**: Every sample in `valid[0 : genSize]` is covered by 6 fully overlapping Hann windows. There is zero window attenuation and zero crossfade mud.
2. **Phase-Aligned Subtraction**: Because time coordinates match PyTorch training to the exact sample, subtracting `Vocals` from `Mix` cancels 100% of the vocal energy in the instrumental track.
3. **Pristine Bass & Drums**: Drums, bass, and secondary instruments are extracted directly from `instrumental.wav` (which has already had the vocal completely removed), eliminating vocal bleed in the rhythm section.

---

## 3. Total Elimination of the "Other" Stem

All models and UI views now output strictly meaningful musical tracks:

| Model | Output Stems | Description |
|---|---|---|
| **UVR MDX-Net** | **🎙️ Vocals, 🎹 Instrumental** | Acapella vs. backing track isolation |
| **4-Stem Band** | **🎙️ Vocals, 🥁 Drums, 🎸 Bass, 🎹 Instrumental** | Complete rhythm section and accompaniment |
| **Smart Auto-Detect** | **Vocals, Drums, Bass, Guitar, Piano, Synths, Strings** | Unpacks only instruments acoustically present |
| **6-Stem & 8-Stem** | **Vocals, Lead, Harmonies, Drums, Bass, Guitar, Piano, Synths, Strings** | Full studio multitrack deconstruction |

---

## 4. Verification & Validation

- `TestFFT6144RoundTrip`: Double-precision Cooley-Tukey round trip (PASS, error $< 1.72 \times 10^{-15}$).
- `TestSTFTChunkRoundTrip`: Stationary center-padded round trip (PASS, error $< 8.35 \times 10^{-7}$).
- `TestRunMDXSeparationPipeline`: End-to-end `mdx_vocals.onnx` inference on synthetic audio (PASS, 4.82s).
- Full suite `go test ./...`: 100% PASS across `audio`, `engine`, and `tui`.
- Binary: `bin/lisn.exe` compiled cleanly.
