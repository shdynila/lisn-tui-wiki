---
title: "LISN-TUI: Multi-Window Workspace Navigation & Background Stem Separation Architecture"
date: 2026-10-23
tags: ["architecture", "audio", "golang", "tui", "bubbletea", "navigation", "multitasking"]
---

# LISN-TUI: Multi-Window Workspace Navigation & Background Stem Separation Architecture

## 1. Executive Summary

As **LISN-TUI** evolved from a simple audio stem separation tool into a full-featured terminal digital audio workstation (DAW) with real-time multi-track mixing, acoustic instrument detection, hardware diagnostics, and mixdown rendering, the original linear screen transition model proved limiting. Users needed the freedom to move between workspaces without losing their place, cancel running processes unintentionally, or memorize hidden commands.

This architectural enhancement introduces a comprehensive **Multi-Window Workspace System** across LISN-TUI:
1. **Interactive Window Tabs in Header**: Real-time visible tab indicators `[1] ✂️ Splitter`, `[2] 🎚️ Mixer`, `[3] 📚 Library`, `[4] 🩺 Doctor`, `[5] 📜 Logs`, and `[6] 💾 Export` with glowing active badges and running job pulses.
2. **Dedicated Direct Shortcuts**: Instant jump via `F1`-`F6`, direct numbers `1`-`6` (outside mixer), and mnemonic keys (`s`, `m`, `c`, `l`, `e`).
3. **Bidirectional Navigation**: Next/previous window cycling via `[` / `]`, `Ctrl+n` / `Ctrl+p` (tmux-style), `Ctrl+Left` / `Ctrl+Right`, and breadcrumb history back navigation via `Esc` / `Backspace`.
4. **Floating Window Manager (`w`)**: Interactive popup workspace switcher dialog.
5. **Non-Blocking Background Separation**: Seamlessly leaves the Splitter while ONNX inference continues in the background, continuously streaming logs and updating progress indicators until complete.
6. **Studio Mixer Standby View**: Elegant, informative standby state when no active audio session is loaded.

---

## 2. Window Decomposition & Layout

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ LISN v1.0.0 [local]  │ [1] ✂️ Splitter* │ [2] 🎚️ Mixer │ [3] 📚 Library │ [4] 🩺 Doctor │ [5] 📜 Logs │  │
│ <F1-F6/w>:windows  <[/]>:nav  <:>:cmd  <?>:help  <Enter>:split  <q>:quit                           │
├────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                    │
│                                       ACTIVE WORKSPACE                                              │
│                                                                                                    │
│  [Window 1: Splitter]       [Window 2: Studio Mixer]      [Window 3: Library / Cache]              │
│  • File / YouTube URL       • Multi-Track Audio Engine    • Cached Songs & Stems                   │
│  • Model Selection Pills    • Solo / Mute / Faders        • Acoustic Instrument Detective          │
│  • Live Separation Monitor  • Waveform Visualizer & Loop  • Fast Search & Live Filter (/)          │
│                                                                                                    │
│  [Window 4: System Doctor]  [Window 5: System Logs]       [Window 6: Audio Exporter]               │
│  • ONNX Runtime & DirectML  • Live Neural Net Logs        • Master Mixdown Rendering               │
│  • Hardware Accelerators    • Audio Driver Diagnostics    • High-Fidelity WAV Stems                │
│                                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Window Specifications

| Window ID | Name | Icon | Direct Key | Function Key | Role |
| :--- | :--- | :---: | :---: | :---: | :--- |
| `WindowHome` | **Home Hub** | `🏠` | `0` / `h` | `Esc` | Primary Main Screen & landing dashboard opening first on launch with live telemetry & selectable cards |
| `WindowSplitter` | **Splitter** | `✂️` | `1` / `s` | `F1` | Neural stem separation setup, YouTube/file input, model selection & live progress monitor |
| `WindowMixer` | **Studio Mixer** | `🎚️` | `2` / `m` | `F2` | Sample-accurate multi-track playback, volume faders, peak VU meters, looping & speed controls |
| `WindowLibrary` | **Library** | `📚` | `3` / `c` | `F3` | Saved separations cache, acoustic instrument detective preview, instant playback, deletion |
| `WindowDoctor` | **Doctor** | `🩺` | `4` / `d` | `F4` | Native ONNX Runtime engine status, hardware acceleration checks, FFmpeg readiness |
| `WindowLogs` | **Logs** | `📜` | `5` / `l` | `F5` | Real-time streaming log engine with autoscroll, scrollback, and clear buffer |
| `WindowExport` | **Export** | `💾` | `6` / `e` | `F6` | Render custom stereo mixdown or export isolated lossless WAV stems to disk |

---

## 3. Core Architectural Mechanisms

### 3.1 Primary Main Screen & Startup Routing (`ViewHome`)

When launching `lisn-tui` interactively (without CLI arguments), the application opens into `ViewHome` first thing:
- **ASCII Brand & Live Telemetry**: Dynamic neon header displaying active ONNX Runtime version, DirectML/CUDA status, and cached song library counts.
- **Interactive Workspace Cards**: Rich, selectable cards for all six workspaces with high-contrast cursor indicators (`❯ [1] ✂️  Stem Splitter`).
- **Direct Launch & Smooth Navigation**: Users can press `Enter` to launch the highlighted workspace, hit `1`-`6` for instantaneous jump, or use `j`/`k`/`↑`/`↓` to browse features.
- **CLI Bypassing**: When executed with a target file (e.g. `lisn song.mp3`), the engine intelligently fast-tracks directly to `ViewDrop` (the Splitter input) with the file pre-populated.

### 3.2 Non-Blocking Background Separation Loop

Previously, separation events were only received when `m.currentView == ViewSeparating`. Switching views caused progress events to be dropped and the listener channel loop to stall.

In the new architecture:
- `engine.ProgressEvent` is decoupled from view rendering and handled directly in `AppModel.Update()` at the root level.
- Background jobs update `m.splitView` state and append to `m.logsView` irrespective of which window the user is currently viewing.
- The tab bar dynamically reflects running jobs via `[1] ✂️ Splitter (35%)` with an amber pulse badge.
- If separation completes while the user is inspecting the Library or Doctor, a completion log is recorded and the mixer is primed without violently stealing user focus.

### 3.3 Bidirectional Navigation & History Stack

To allow seamless back-and-forth movement:
- **Sequential Cycling**: `[` and `]` (outside the studio player where `[`/`]` set loop points), `Ctrl+n` / `Ctrl+p`, and `Ctrl+Left` / `Ctrl+Right` advance or retreat sequentially across the workspaces.
- **History Stack (`viewHistory []ScreenView`)**: Breadcrumb stack that records up to 30 past navigation steps. Pressing `Esc` or `Backspace` unwinds the stack to return to the exact prior view without cyclical traps.
- **Floating Window Picker (`w`)**: Opens an interactive modal allowing cursor navigation (`↑`/`↓`, `j`/`k`, `Enter`) or single-key jump (`0`-`6`).
- **Universal Return Home (`0` / `h`)**: Instantly returns to the Main Screen from any workspace.

### 3.4 Studio Mixer Standby State

Rather than refusing navigation or causing nil dereferences when no song has been loaded yet, `PlayerModel` renders a stylized **Standby View**. It presents contextual instructions directing users to separate a new song (`[1] Splitter`) or choose a cached track from disk (`[3] Library`).

---

## 4. Verification & Validation

The test suite in `pkg/tui/app_test.go` verifies all navigation pathways:
- `TestAppModelInit`: Validates `ViewHome` defaults as the first screen on launch, while validating that providing a file via CLI opens `ViewDrop`.
- `TestHomeScreenNavigation`: Validates interactive cursor navigation (`j`/`k`), Enter selection, direct jump (`1`-`6`), and returning home with `Esc`/`h`.
- `TestWindowWorkspaces`: Validates `F1`-`F6`, number keys `1`-`6`, mnemonic shortcuts, `[`/`]` cycling, `Ctrl+p` player cycling, history back via `Backspace`, floating modal activation via `w`, and command palette commands (`:splitter`, `:win`).
- `TestWindowBackgroundSeparation`: Validates that switching to `ViewLogs` while ONNX inference is actively running does not drop progress events, correctly updates `splitView.percentage`, maintains tab badges, and returns to `ViewSeparating` when `F1` is pressed.
