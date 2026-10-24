---
title: "Devlog: Pure Acoustic Stem Isolation & Zero-Friction Auto-Unpack"
date: 2026-10-24T12:05:00+02:00
tags: ["lisn-tui", "devlog", "audio", "onnx", "acoustic-detective"]
draft: false
---

## Summary of Changes

Today we tackled the root causes behind vocal leakage on identified stems and eliminated the manual unpack requirement from the LISN-TUI studio workflow:

1. **Elimination of Fake Vocal Splitting & Multiplication**:
   - Diagnosed why vocal bleed was perceived on identified stems: stereo reverb on solo singing was being misclassified as "Harmonies" (`StemHarmonyVocals`), which also generated "Lead Vocals" (`StemLeadVocals`).
   - Having `Vocals`, `Lead Vocals`, and `Harmonies` unmuted simultaneously in the mixer tripled the vocals and leaked vocal energy into solos.
   - Refactored `IdentifyPresentSubStems` to focus strictly on genuine musical instruments inside `instrumental.wav`, keeping neural `Vocals` singular, pristine, and uncorrupted.

2. **Source Invariant for Non-Vocal Stems**:
   - Stems such as `Drums`, `Bass`, `Synthesizers`, `Guitar`, `Piano`, and `Strings` are now strictly enforced to use `instrumental.wav` as input.
   - Removed any fallback to the original mixed audio, guaranteeing mathematically zero vocal bleed.

3. **Automated End-to-End Unpacking**:
   - Removed the need for the manual `[u] Unpack` step.
   - During `Separate()` for `ModelONNXAuto`, the acoustic detective immediately identifies present instruments and extracts them directly into the session.
   - Updated the mixer toolbar and help shortcuts to replace `[u] Unpack` with `[a] Unmute All`.

4. **Cache Sanitation**:
   - Cleaned up outdated mock stems across cache sessions and verified with `TestUnpackAcidArab` that *Acid Arab - Stil* extracts clean Drums, Bass, and Synthesizers with zero vocal bleed.
