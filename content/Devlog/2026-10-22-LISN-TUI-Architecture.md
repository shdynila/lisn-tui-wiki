---
title: "Building LISN-TUI: Rewriting an Audio Stem Splitter in Go (100% Python-Free ONNX)"
date: 2026-10-22
tags: ["devlog", "audio", "golang", "tui", "bubbletea", "onnx", "dsp"]
---

# Building LISN-TUI: Rewriting an Audio Stem Splitter in Go (100% Python-Free ONNX)

Today we created and evolved **`lisn-tui`**, a brand new private repository and terminal-based audio workstation inspired by [LISN StemSplitter](https://github.com/EdiKrasniqi11/lisn-vst). The goal was to take the core value proposition of LISN—splitting songs into stems and playing them in sync—and rewrite the entire stack in **pure Go** using a modern **Terminal User Interface (TUI)**.

## Dropping Python & Moving to Native ONNX Runtime

During the initial design phase, the project utilized Demucs wrappers via Python. However, because `facebookresearch/demucs` has been archived and Python virtualenv management introduces significant runtime friction, we migrated the core inference engine to **Approach B: Native Go via ONNX Runtime (`github.com/yalue/onnxruntime_go`)**.

### Benefits of the ONNX Migration
1. **Zero Python Dependency**: No Python interpreters, no conda, no pip dependency conflicts, and no PyTorch wheel mismatches.
2. **Modern Neural Network Architectures**: Replaced legacy Demucs with state-of-the-art models:
   - **Studio 8-Stem Deconstruction (`onnx_all`)**: Fully deconstructs audio into 8 distinct isolated tracks (Lead Vocals, Harmonies, Drums, Bass, Guitars, Pianos, Synthesizers, Pads & Strings) with 0% undefined "Other".
   - **Studio 6-Stem Band (`onnx_6stem`)**: Isolates Guitars, Pianos, and Synthesizers as dedicated channels alongside Vocals, Drums, and Bass.
   - **MDX-B 4-Stem ONNX (`onnx_4stem`)**: Standard 4-stem band separation (Vocals, Drums, Bass, Other).
   - **UVR MDX-Net Vocals (`mdx_vocals`)**: Kim/UVR architecture for rapid vocal and instrumental separation.
   - **Mel-Band RoFormer Vocals (`roformer_vocals`)**: SOTA vocal isolation with unmatched acoustic fidelity.
3. **Automated Runtime & Weights Management**: Automatic detection of system `onnxruntime.dll` / `libonnxruntime.so`, or on-demand fetch of official Microsoft ONNX Runtime binaries, with model weights downloaded seamlessly from Hugging Face.

---

## Technical Highlights

### 1. Pure Go Audio Playback & Mixer (`pkg/audio`)
Instead of wrapping miniaudio or PortAudio with CGO (which breaks easy cross-compilation), we built a pure Go multi-track mixing engine on top of `github.com/gopxl/beep`:
- Sample-accurate lockstep streaming across 4 to 8 stems.
- Dynamic Volume, Mute, and Solo matrices.
- Soft-knee tanh saturation limiter to avoid harsh clipping when boosting multiple tracks.
- High-precision 16/24-bit stereo WAV export renderer.

### 2. Real-Time ASCII Waveform & VU Meters
Using a 30ms ticker loop decoupled from the audio playback thread, `lisn-tui` computes real-time RMS/peak envelopes and translates them into:
- Bouncing ANSI level meters (` ▂▃▄▅▆▇█`) per track and for the master output bus.
- Full-width ASCII waveform overview showing playhead position (`▼`) and loop boundaries (`├` and `┤`).

### 3. Native ONNX AI Separation Engine (`pkg/engine`)
- Direct execution via `github.com/yalue/onnxruntime_go` (C-API bindings).
- **Native YouTube Audio Stream Importer**: Paste any `youtube.com/watch`, `youtu.be`, or `shorts/` link directly in place of a file path. Standalone `yt-dlp` and `ffmpeg` extract high-quality audio streams without Python dependencies.
- **Unified Cache Resolution**: URLs are canonicalized to video IDs, and the cache manager resolves both raw URLs and extracted audio files (`downloads/<videoID>.wav`). Stems open instantaneously on subsequent runs with zero inference lag.
- **Direct Event Result Forwarding**: Separation results are passed directly via `ProgressEvent.Result` to eliminate state-reconstruction cache misses.
- Secondary vocal/harmony splitter for isolating lead vocals from backing vocals.

### 4. k9s Command Palette & Workflows
Inspired by Kubernetes' `k9s`, the TUI introduces:
- **`:` Command Prompt**: Switch between `:stems`, `:cache`, `:open`, `:doctor`, `:logs`, `:export`, and `:detect`.
- **`d` Describe**: Inspect detailed audio metadata and launch the Acoustic Instrument Detective.
- **`/` Real-Time Filter**: Instant substring filtering across cached separations.
- **`l` Logs Viewer**: Live scrollable output with line numbers and autoscroll toggles.
- **`ctrl-d` / `x` Delete**: Safe modal confirmation before deleting cached separations.

### 5. Acoustic Instrument Detective & Stem Decomposition (`lisn detect`)
To solve the mystery of what exists inside the standard 4-stem "Other" and "Vocals" tracks, we engineered an acoustic feature inspection engine (`pkg/engine/detective.go`):
- **Feature Extraction**: Measures multi-band spectral energy ratios (Sub 20-80Hz, Low 80-250Hz, Low-Mid 250-800Hz, Mid 800-2500Hz, High-Mid 2.5-6kHz, High 6-16kHz), Crest Factor (transient dynamic punch vs. sustained pad/organ decay), zero-crossing rates, and onset transient density.
- **Mid/Side Stereo Staging**: Determines stereo width to differentiate center-focused lead elements from wide spatial reverb washes.
- **Deconstructs "Other"**: Identifies Grand Piano, Rhodes/Wurlitzer, Electric Guitars, Acoustic Guitars, Synth Leads & Arps, Ambient Synth Pads/Strings, and Brass/Horns with confidence meters.
- **Vocal & Rhythm Breakdown**: Identifies Lead Vocals, Backing Harmonies, Reverb Tails, Kick Drums, Snares, Hi-Hats, Toms, Sub-Bass/808s, and Electric/Synth Basses.
- **Integrated Surfaces**: Accessible via the dedicated `lisn detect [file/dir/url]` CLI command, automatically appended to `lisn split`, and embedded into the TUI Describe modal (<kbd>d</kbd> / `:detect`).

---

## Repository
- **Repository**: `shdynila/lisn-tui` (Private GitHub repository)
- **Local Location**: `c:\Users\nil\Desktop\prjcts\lisn-tui`
- **Binary**: `lisn.exe` / `lisn`
