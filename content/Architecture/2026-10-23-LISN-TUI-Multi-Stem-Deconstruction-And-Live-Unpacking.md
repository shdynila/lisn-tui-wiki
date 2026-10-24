---
title: "Architecture: LISN-TUI Multi-Stem Deconstruction & Live Studio Unpacking"
date: 2026-10-23
tags: ["audio", "architecture", "lisn-tui", "dsp", "onnx", "multi-stem", "golang"]
---

# LISN-TUI Multi-Stem Deconstruction & Live Studio Unpacking

## Overview
LISN-TUI previously provided high-accuracy 2-stem isolation (`Vocals` vs `Instrumental/Backing`) powered by UVR MDX-Net `demix_base` STFT reflection inference. This release expands the architecture into a full **Multi-Stem Studio Deconstruction Engine**, enabling users to unpack song-specific instruments—Drums, Bass, Guitars, Pianos/Keys, Synthesizers, Pads/Strings, and Harmonies—both during separation and live during playback.

---

## 1. Smart Multi-Stem Studio as Default Model (`ModelONNXAuto`)

In `pkg/engine/types.go` and `pkg/tui/view_drop.go`, `ModelONNXAuto` is promoted to index `0` as the default model:

```go
func AvailableModels() []ModelType {
    return []ModelType{
        ModelONNXAuto,       // ⚡ Smart Multi-Stem Studio: Dynamic Present Stems (Default)
        ModelONNX6Stem,      // Studio 6-Stem: Vocals, Drums, Bass, Guitar, Piano, Synths
        ModelONNXAll,        // Studio 8-Stem: Lead, Harmonies, Drums, Bass, Guitar, Piano, Synths, Strings
        ModelONNX4Stem,      // 4-stem ONNX: Vocals, Drums, Bass, Instrumental
        ModelMDXVocals,      // UVR MDX-Net: Vocals & Instrumental (2-Stem)
        ModelRoformerVocals, // Mel-Band RoFormer: SOTA Vocal Isolation
    }
}
```

When separating with `ModelONNXAuto`:
1. **Base Neural Separation**: UVR MDX-Net isolates `vocals.wav` and subtracts it from the mix to form pristine `instrumental.wav` ($0$ vocal bleed).
2. **Acoustic Detective Analysis**: `IdentifyPresentSubStems` analyzes `instrumental.wav` and `vocals.wav` using crest factor, spectral centroid, zero-crossing rate, and mid/side stereo spread.
3. **Multi-Stem Extraction**: Renders isolated stems for:
   - 🥁 **Drums & Percussion**: Transient onset shaping, kick punch (75 Hz), and cymbal sizzle (10.5 kHz).
   - 🎸 **Bass**: Sub-lowpass ($<220$ Hz, 2-pole) and low-end resonance (65 Hz).
   - 🎸 **Guitar**: Mid-band harmonic drive (160 Hz - 4.8 kHz) with notch filtering at 80 Hz to eliminate kick thumps.
   - 🎹 **Piano & Keys**: Soundboard warmth (120 Hz - 4.2 kHz) and stereo spread.
   - 🎛️ **Synthesizers**: Electronic bite (350 Hz - 9.5 kHz) and fast attack dynamics.
   - 🌌 **Pads & Strings**: Stereo difference side-channel ($c_0 - 0.75 \cdot c_1$) ambient wash.
   - 👥 **Harmonies & Lead Vocals**: Extracted from `vocals.wav` into centered lead vs stereo backing vocals.

---

## 2. Live Stem Unpacking in Studio Mixer (`u` Hotkey)

If a user plays a 2-stem track or wishes to deconstruct the instrumental backing into individual tracks:
1. Pressing `u` in the Player View (`view_player.go`) emits `unpackStemsMsg{stemDir: m.stemDir}`.
2. The engine executes `UnpackMultiStems` in a non-blocking background goroutine:
   ```go
   extracted, err := m.runner.UnpackMultiStems(ctx, stemDir, nil)
   ```
3. Upon completion, `app.go` receives `unpackedStemsDoneMsg` and registers each new stem into the audio engine using `AddOrReplaceTrack(stemType, stemPath)`.
4. Playhead position and transport playback continue seamlessly with zero interruption or audio drift.

---

## 3. Deterministic Studio Console Channel Ordering

Previously, Go map iteration over `stems` caused random mixer track orders across sessions. `pkg/audio/engine.go` now enforces standard professional DAW channel priority:

| Channel Priority | Stem Type | Description | Default State |
| :--- | :--- | :--- | :--- |
| **Track [1]** | `Lead Vocals` / `Vocals` | Centered lead melodic vocal | Active |
| **Track [2]** | `Harmonies` | Stereo backing vocal doubles | Active |
| **Track [3]** | `Instrumental` | Full backing mix | Muted (if sub-stems present) |
| **Track [4]** | `Drums` | Kick, snare, hats, percussion | Active |
| **Track [5]** | `Bass` | Sub-bass and 808s | Active |
| **Track [6]** | `Guitar` | Acoustic and electric guitars | Active |
| **Track [7]** | `Piano` | Grand piano and keys | Active |
| **Track [8]** | `Synthesizers` | Electronic synths and leads | Active |
| **Track [9]** | `Pads & Strings` | Ambient atmospheres and strings | Active |

When individual sub-stems are present, `StemInstrumental` is initialized as muted so individual instrument faders play cleanly without volume doubling, while remaining instantly available for reference or soloing.

---

## 4. Vocal Peak Normalization & Intro Silence Alert

1. **Vocal Gain Compensation**: In `pkg/engine/mdx.go`, if the isolated vocal track has low peak energy ($< 0.65$), gentle normalization boosts the signal to $0.85$ peak level (up to $+9.5$ dB), ensuring quiet vocal performances are crisp and prominent.
2. **Intro Silence Detection**: `pkg/audio/track.go` scans sample buffers with `FirstSoundSecond()`. When a user solos a track whose audio begins after 3.0 seconds (e.g., songs with a 48-second instrumental intro), the mixer status displays:
   ```
   ⭐ Solo Vocals (Audio begins at 00:48 • press →/l to seek)
   ```
