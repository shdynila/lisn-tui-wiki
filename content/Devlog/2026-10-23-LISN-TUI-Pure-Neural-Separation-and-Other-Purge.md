---
title: "Devlog: LISN-TUI Pure Neural Separation and Purging the 'Other' Stem"
date: 2026-10-23
tags: ["devlog", "audio", "neural-networks", "mdx-net", "onnx", "golang", "dsp"]
---

# Devlog: LISN-TUI Pure Neural Separation and Purging the 'Other' Stem

Today we tracked down and resolved the underlying acoustic separation and vocal bleed issues:

### 1. The Legacy Cache Discovery
When checking the user's cache directory (`$LOCALAPPDATA/lisn-tui/cache`), we uncovered 15 cached song folders dating back to **10/22/2026** with identical byte sizes across all stems. When selecting tracks from "Recent Separations" or repeating URLs, the app was instantly serving the old mock FFmpeg equalizer files created yesterday!
- We purged the legacy cache from disk.
- We updated `verifyStemDir` to automatically invalidate any cached directory that still contains `other.wav`.

### 2. PyTorch `center=True` & The UVR `demix_base` Architecture
In reviewing the official MDX-Net architecture:
- UVR MDX-Net models are trained with `torch.stft(..., center=True, pad_mode='reflect')` with $N_{\text{fft}} // 2 = 3072$ reflection padding.
- Without this 3072-sample center offset, the spectrogram had a 70ms time-shift, which distorted the neural network's receptive field and created a comb filter when calculating $\text{Instrumental} = \text{Mix} - \text{Vocals}$.
- We implemented the exact UVR `demix_base` algorithm in Go: 261,120-sample chunks (`MDX_CHUNK_SIZE`), 3072-sample boundary margins (`MDX_TRIM`), and 254,976-sample stationary blocks (`MDX_GEN_SIZE`). 
- Each chunk discards edge tapers and tiles seamlessly with zero overlap artifacts and sample-exact phase alignment.

### 3. Elimination of "Other"
We permanently purged the `"other"` stem across all separation models. Users now get **🎙️ Vocals** and **🎹 Instrumental / Backing** for 2-stem models, and dedicated, real instrument tracks for multi-stem models (Vocals, Drums, Bass, Guitars, Pianos, Synths, Strings).

All tests in `audio`, `engine`, and `tui` pass, and `lisn.exe` is compiled.
