---
title: "Devlog: Multi-Stem Deconstruction & Live Studio Unpacking"
date: 2026-10-23
tags: ["devlog", "audio", "neural-networks", "lisn-tui", "dsp", "multi-stem", "golang"]
---

# Devlog: Multi-Stem Deconstruction & Live Studio Unpacking

Following the implementation of pristine UVR MDX-Net vocal subtraction, we tackled expanding from 2-stem isolation (`Vocals` and `Instrumental`) to full **multi-stem deconstruction** of the entire musical arrangement:

### 1. The Multi-Stem Expansion
When separating audio, users want to hear more than just the backing track; they want individual faders for every element:
- 🥁 **Drums & Percussion**: Isolated kicks, snares, and cymbals.
- 🎸 **Bass**: Dedicated bassline fader with deep sub-resonance.
- 🎸 **Guitars**: Acoustic strumming and electric leads.
- 🎹 **Pianos & Keys**: Warm acoustic piano resonance and stereo keys.
- 🎛️ **Synthesizers**: Electronic arpeggiators and leads.
- 🌌 **Pads & Strings**: Ambient spatial atmosphere.
- 👥 **Harmonies & Lead Vocals**: Backing vocals separated from center lead vocal.

### 2. Live Unpacking in the Player (`u` Hotkey)
If a user is already playing a 2-stem track (like *Acid Arab - Stil*):
- Pressing `u` ("Unpack Multi-Stems") executes background extraction without interrupting playback.
- The audio engine registers the newly extracted stems in real time using `AddOrReplaceTrack()`.
- The mixer UI automatically expands from 2 channels to 6-8 channels live while the song plays in lockstep synchronization.

### 3. Studio Console Channel Priority & Solo Intro Notice
- We sorted all tracks deterministically: Track [1] is always Vocals, Track [2] Harmonies, Track [3] Instrumental, followed by Drums, Bass, Guitars, Keys, and Synths.
- In *Acid Arab - Stil*, the vocal track begins with 48 seconds of complete silence. We added intro silence detection (`FirstSoundSecond()`) so soloing an intro-silent stem notifies the user:
  `⭐ Solo Vocals (Audio begins at 00:48 • press →/l to seek)`.
- Quiet vocal tracks are gently normalized up to 0.85 peak so vocals are clear and prominent.

Tested and verified 100% in pure Go + Microsoft ONNX Runtime C-API.
