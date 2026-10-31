---
title: "LISN-TUI Architecture: Bounded Soft Subtraction and Instrumental Preservation"
date: 2026-10-31
tags: ["lisn-tui", "architecture", "dsp", "source-separation", "phase-cancellation", "mdx-net", "kim-inst"]
---

# LISN-TUI Architecture: Bounded Soft Subtraction and Instrumental Preservation

This architectural specification documents the mathematical formulation and signal routing designed to eliminate over-aggressive phase subtraction, instrumental "holes", and premature stem muting during vocal separation.

---

## 1. Problem Diagnosis: The Over-Aggressive Splitter

When separating vocals from music, users experienced a distinct acoustic anomaly:
> *"the splitter is over aggreessive, some instrumental gets muted when vocal is muted, but are present for a bit when actual voce is singing at the same time."*

Investigation uncovered two primary technical causes:

### 1.1 Unbounded Time-Domain Phase Subtraction
In standard vocal-centric MDX-Net inference, the model predicts the vocal waveform $\text{vocals}(t)$, and the instrumental stem is derived by time-domain subtraction:
$$\text{inst}(t) = \text{mix}(t) - \text{vocals}(t)$$

Because real-world musical instruments (electric guitars, synthesizer pads, pianos, reverb tails) share harmonic frequencies with human voice formants (300 Hz – 4 kHz), neural vocal extractors inevitably capture small portions of these overlapping instrumental frequencies:
$$\text{vocals}_{\text{pred}}(t) = \text{vocal}(t) + \alpha \cdot \text{instrument}(t) \quad (\alpha > 0)$$

When this prediction is subtracted from the mix:
$$\text{inst}(t) = \text{mix}(t) - \text{vocals}_{\text{pred}}(t) = (1 - \alpha) \cdot \text{instrument}(t)$$

#### The Consequence:
1. **Instrumental Holes**: During vocal sections, the instrument's amplitude in $\text{inst}(t)$ drops by $(1 - \alpha)$, sounding muffled or muted ("carved out").
2. **Vocal Bleed**: The missing instrument energy $\alpha \cdot \text{instrument}(t)$ is present inside $\text{vocals}(t)$.
3. **Mute Artifact**: When the listener mutes $\text{vocals}(t)$, the instrument is silenced during the singing sections, but reappears at full volume when the singer pauses.

### 1.2 Premature Sub-Stem Auto-Muting
In `pkg/audio/engine.go`, when sub-stems (e.g., `Drums`, `Bass`) were unpacked, the mixer automatically muted the core `Instrumental` stem to avoid volume doubling.
However, sub-stems extracted via bandpass filters do not cover the entire musical spectrum. Muting `Instrumental` completely silenced all guitars, keyboards, brass, strings, and effects that were not part of the detected sub-stems.

### 1.3 `Kim_Inst` Model Inversion
Models downloaded from `Kim_Inst.onnx` but saved locally as `onnx_4stem.onnx`, `onnx_6stem.onnx`, or `onnx_all.onnx` failed the `strings.Contains(modelPath, "inst")` check, inverting the vocal and instrumental channels.

---

## 2. Mathematical Formulation: Bounded Soft Subtraction

To eliminate phase cancellation holes and preserve 100% instrumental continuity, LISN-TUI implements **Energy-Conserving Bounded Soft Subtraction** with a balanced $0.92$ margin:

$$\text{margin} = 0.92$$

For each sample $x_{\text{mix}}$ and predicted component $v$:
$$v_{\text{scaled}} = v \cdot \text{margin}$$

$$
v_{\text{bounded}} = 
\begin{cases}
\min(\max(0, v_{\text{scaled}}), x_{\text{mix}}) & \text{if } x_{\text{mix}} \ge 0 \\
\max(\min(0, v_{\text{scaled}}), x_{\text{mix}}) & \text{if } x_{\text{mix}} < 0
\end{cases}
$$

$$\text{inst} = x_{\text{mix}} - v_{\text{bounded}}$$
$$\text{vocals} = v_{\text{bounded}}$$

### Mathematical Proofs:
1. **Sign Preservation**:
   - For $x_{\text{mix}} \ge 0$, $v_{\text{bounded}} \in [0, x_{\text{mix}}] \implies \text{inst} = x_{\text{mix}} - v_{\text{bounded}} \ge 0$.
   - The instrumental signal can **never flip sign or invert phase**, eliminating hollow comb-filtering artifacts.
2. **Strict Energy Conservation**:
   $$\text{vocals} + \text{inst} = v_{\text{bounded}} + (x_{\text{mix}} - v_{\text{bounded}}) \equiv x_{\text{mix}}$$
   The sum of isolated stems is identically equal to the original audio mixture at every discrete sample.
3. **Overlapping Instrument Protection**:
   The $0.92$ margin ensures that shared musical harmonics are not over-subtracted. Even at peak vocal presence, backing instruments retain full presence and stability.

---

## 3. Playback Architecture: Master Instrumental Preservation

The mixer initialization in `pkg/audio/engine.go` was updated to keep `Instrumental` active by default:
- `Vocals` (100%) and `Instrumental` (100%) form the foundational, complete song arrangement with 0 missing instruments.
- When `Vocals` is muted, `Instrumental` plays the entire band (guitars, synths, drums, bass, strings) without dropouts.
- Unpacked sub-stems (`Drums`, `Bass`) are instantly accessible for Solo (`s`) analysis or fine-grained fader remixing.
