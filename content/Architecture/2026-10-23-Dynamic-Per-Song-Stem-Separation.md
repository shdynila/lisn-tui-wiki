---
title: "LISN-TUI: Dynamic Per-Song Stem Identification vs. 4-Stem Limitation"
date: 2026-10-23
tags: ["architecture", "audio", "stem-separation", "acoustic-detective", "onnx", "golang", "dsp"]
---

# LISN-TUI: Dynamic Per-Song Stem Identification vs. 4-Stem Limitation

## 1. Why Is Traditional AI Audio Separation Limited to 4 Stems?

Across the audio AI industry (Demucs, Spleeter, OpenUnmix, standard MDX-B), almost every tool defaults to **4 stems**:
1. `vocals`
2. `drums`
3. `bass`
4. `other`

### The Root Cause: The Academic Dataset Bottleneck (MUSDB18)

1. **Training Dataset History**: In 2018, the Signal Separation Evaluation Campaign (SiSEC) established **MUSDB18** as the standardized public benchmark for audio source separation. MUSDB18 provided multi-tracks strictly partitioned into 4 stems: `vocals`, `drums`, `bass`, and `other`.
2. **The "Other" Catch-All Sink**: Any instrument that wasn't a singing human, a drum strike, or a bass frequency was labeled as `"other"`. This meant guitars, acoustic pianos, analog synths, violin sections, trumpet stabs, flute runs, and sound effects were all collapsed into one muddy audio file.
3. **Model Fixed Output Topology**: Convolutional/U-Net and Transformer architectures (like Demucs or MDX) produce output tensors configured for a fixed number of audio channels (e.g., 4 stems $\times$ 2 stereo channels). Standard models simply do not have output heads for instruments outside their fixed training targets.

---

## 2. The LISN-TUI Solution: Dynamic Song-Specific Stem Identification

In real music, **every song is unique**:
- An acoustic folk ballad might have **Lead Vocals, Backing Harmonies, Acoustic Guitar, and Cello** (zero drums, zero electric bass).
- An 80s synth-pop anthem might have **Vocals, 808 Drums, Synth Bass, Synthesizer Leads, and Ambient Pads**.
- A rock track might feature **Guitars, Bass, Drums, and Piano**, but no synthesizers or strings.

Forcing every song into `Vocals / Drums / Bass / Other` produces empty/useless tracks or traps the best musical parts inside `"other"`.

To solve this, **LISN-TUI implements an Acoustic Detective + Dynamic Stem Deconstruction Architecture** (`ModelONNXAuto`):

```
                                  [ Input Audio Track ]
                                            │
                                            ▼
                           [ ONNX Neural Separation Base ]
                             ├── 🎙️ Vocals
                             ├── 🥁 Drums
                             ├── 🎸 Bass
                             └── 🎼 Other
                                            │
                                            ▼
                        [ Acoustic Detective Feature Extractor ]
                          • Spectral Energy Bands (Sub, Low, Mid, High)
                          • Crest Factor (Transient Punch vs. Pad Tail)
                          • Stereo Phase & Width (L/R Spread vs. Center)
                          • Harmonic Centroid & Onset Density
                                            │
                     ┌──────────────────────┴──────────────────────┐
                     ▼                                             ▼
          [ Vocals Analysis ]                             [ "Other" Analysis ]
           • Stereo Width > 0.38?                          • Mid-Band Ratio > 0.14 & Crest > 5.5? ➔ 🎸 Guitar
             ➔ 🌟 Lead Vocals (Center)                    • Low-Mid Body > 0.15 & Stereo Spread? ➔ 🎹 Piano
             ➔ 👥 Backing Harmonies (Sides)               • High Onset Density (>4/s) & High-Mid? ➔ 🎛️ Synths
                                                           • Ultra-Wide Stereo & Low Crest? ➔ 🌌 Pads & Strings
                                                           • Mid Attack Stabs (>20%)? ➔ 🎺 Brass & Horns
                                            │
                                            ▼
                     [ Dynamic Per-Song Multi-Track Mix Console ]
                       (Only genuine, acoustically present stems)
```

---

## 3. How Dynamic Auto-Detect Operates in LISN-TUI

1. **Acoustic Presence Thresholding ($\ge 0.78$ Confidence)**:
   The Acoustic Detective inspects RMS energy, frequency centroids, stereo width, and crest factor. Only instruments genuinely detected above confidence thresholds are queued for extraction.
2. **Deconstruction & Catch-All Elimination**:
   - If Guitar, Piano, Synths, Strings, or Brass are unpacked from `other.wav`, the generic `other.wav` is removed from the active mixer so audio does not duplicate.
   - If Harmonies are detected in `vocals.wav`, LISN-TUI unpacks both `🌟 Lead Vocals` and `👥 Harmonies`, eliminating generic `vocals.wav`.
3. **Scalable Multi-Track Mixer (`pkg/audio/engine.go`)**:
   - The Go audio engine accepts an arbitrary stem map `map[engine.StemType]string`.
   - Whether a track unpacks into 3 stems or 9 stems, the Studio Mixer dynamically provisions individual faders, mute/solo buses, and peak VU meters.
4. **Default Mode**:
   `ModelONNXAuto` is now the primary, default model in `AvailableModels()` and displayed with the `⚡ Smart Dynamic (Song-Specific Stems)` badge in the Splitter.

---

## 4. Summary of Supported Stems

| Stem Type | Icon | Frequency & Acoustic Signature | Origin |
| :--- | :---: | :--- | :--- |
| `StemLeadVocals` | 🌟 | Centered mono focus, 200 Hz - 4.5 kHz formant energy | Deconstructed from Vocals |
| `StemHarmonyVocals`| 👥 | Stereo L/R spread ($>0.38$), choral layering | Deconstructed from Vocals |
| `StemVocals` | 🎙️ | Full vocal frequency range | Base Neural Pass |
| `StemDrums` | 🥁 | High crest factor, 40 Hz kick sub to 16 kHz cymbal shimmer | Base Neural Pass |
| `StemBass` | 🎸 | Solid center mono, 30 Hz - 450 Hz fundamentals | Base Neural Pass |
| `StemGuitar` | 🎸 | Mid-band harmonic drive (160 Hz - 5.2 kHz), pick transients | Unpacked from Other |
| `StemPiano` | 🎹 | Low-mid body warmth (120 Hz - 4.2 kHz), keyboard stereo image | Unpacked from Other |
| `StemSynths` | 🎛️ | Fast transient onset density ($>4.0$/s), 350 Hz - 9 kHz | Unpacked from Other |
| `StemStrings` | 🌌 | Ultra-wide stereo wash ($>0.40$), sustained pads (low crest) | Unpacked from Other |
| `StemBrass` | 🎺 | Dynamic brass bite (220 Hz - 5.5 kHz), punchy accent stabs | Unpacked from Other |
