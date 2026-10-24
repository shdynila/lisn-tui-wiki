---
title: "Breaking Past the 4-Stem Bottleneck: Dynamic Per-Song Stem Isolation"
date: 2026-10-23
tags: ["lisn", "tui", "audio", "stem-separation", "acoustic-detective", "devlog"]
---

# Breaking Past the 4-Stem Bottleneck: Dynamic Per-Song Stem Isolation

A common question in AI audio separation is: **Why are almost all stem splitters hardcoded to 4 stems? And can't we isolate whatever instruments actually exist in a specific song?**

Today we made **Dynamic Auto-Detect (`ModelONNXAuto`)** the default neural separation pipeline in **LISN-TUI**, breaking free of the rigid 4-stem limitation.

---

## 1. Why Was Everything Stuck on 4 Stems?

1. **The MUSDB18 Benchmark**: In 2018, academic researchers established the MUSDB18 dataset as the de-facto standard for training neural networks (Demucs, Spleeter, MDX-Net). MUSDB18 only had 4 isolated stems: `vocals`, `drums`, `bass`, and `other`.
2. **The "Other" Black Hole**: Any instrument that wasn't a vocal, drum, or bass got thrown into the "Other" bucket. Guitars, grand pianos, brass sections, synthesizer leads, and orchestral strings were mushed together.
3. **Fixed Neural Topologies**: Standard neural networks have fixed output heads (4 channels). They have no way to dynamically know if a track has an acoustic guitar or a saxophone.

---

## 2. How LISN-TUI Solves This: Dynamic Per-Song Stems

Rather than forcing every piece of music into an arbitrary 4-stem box, LISN-TUI combines neural inference with an **Acoustic Detective**:

1. **Acoustic Fingerprinting**: The detective measures frequency bands (sub, low, low-mid, mid, high-mid, high), stereo image spread, crest factor (percussive attack vs. sustained tail), and transient onset density.
2. **Dynamic Instrument Identification**:
   - Detects whether `other.wav` contains **Guitars** (mid-band drive & pick dynamics), **Pianos** (lower-mid body & stereo keyboard spread), **Synthesizers** (high onset attack density), **Pads & Strings** (ultra-wide stereo wash & sustained resonance), or **Brass & Horns** (dynamic mid-band stabs).
   - Detects whether `vocals.wav` contains **Centered Lead Vocals** and **Stereo Backing Harmonies**.
3. **Selective Deconstruction**:
   - If an instrument is detected with $\ge 0.78$ confidence, it is isolated into its own stem.
   - If a song doesn't have an instrument (e.g. no guitar in an electronic track), LISN-TUI does not create useless, empty channels!
   - Generic catch-alls (`other.wav` or generic `vocals.wav`) are discarded so there is zero audio doubling.
4. **Adaptive Studio Mixer**:
   - The Studio Mixer dynamically adjusts its faders, solo, mute, and VU meters to whatever number of stems was identified (from 2 up to 8+ tracks).

`ModelONNXAuto` is now the default mode in LISN-TUI with the badge `⚡ Smart Dynamic (Song-Specific Stems)`.
