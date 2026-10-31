---
title: "Devlog: Eliminating Over-Aggressive Separation and Preserving Instrumental Fullness"
date: 2026-10-31
tags: ["lisn-tui", "devlog", "dsp", "audio", "subtraction", "mdx-net", "kim-inst"]
---

# Devlog: Eliminating Over-Aggressive Separation and Preserving Instrumental Fullness

## Context & Motivation
Following the audio headroom fixes, a nuanced acoustic issue was reported during vocal/instrumental playback:
> *"also the splitter is over aggreessive, some instrumental gets muted when vocal is muted, but are present for a bit when actual voce is singing at the same time."*

Investigating this revealed that raw time-domain subtraction ($\text{inst} = \text{mix} - \text{vocals}$) was acting as an accidental comb filter, subtracting overlapping musical instruments (guitars, synths, reverb tails) whenever the singer was active. In addition, `NewEngine` was auto-muting the master `Instrumental` stem whenever sub-stems were unpacked, leaving unextracted instruments stranded.

---

## What We Changed

### 1. Bounded Soft Subtraction (`subMargin = 0.92`)
- In `pkg/engine/mdx.go`, replaced raw subtraction with bounded soft subtraction.
- Constrained vocal magnitude to $\le \text{mix}$ with identical sign, preventing phase inversion and eliminating "holes" in the music.
- Guaranteed perfect energy conservation: $\text{vocals}(t) + \text{inst}(t) \equiv \text{mix}(t)$.

### 2. Master Instrumental Preservation
- In `pkg/audio/engine.go`, removed the automatic muting of `Instrumental` when sub-stems exist.
- When `Vocals` is muted, the listener now hears the complete, rich musical arrangement (guitars, keyboards, synths, effects) without dropouts.

### 3. Model Detection Fix for `Kim_Inst`
- Updated `isInstModel` in `pkg/engine/mdx.go` to recognize `onnx_4stem`, `onnx_6stem`, and `onnx_all`, ensuring instrumental-primary neural models are never inverted.

### 4. Cache Engine Bump (`v4`)
- Updated `CacheEngineVersion = "v4"` in `pkg/engine/cache.go` so legacy over-aggressive stems are superseded by full-bodied audio.

---

## Verification
- Ran full test suite (`go test ./...`) with 100% pass rate.
- Verified audio engine binary builds cleanly.
- Triggered automated release workflow for `v0.2.2`.
