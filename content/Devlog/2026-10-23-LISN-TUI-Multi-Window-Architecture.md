---
title: "LISN-TUI: Adding Multi-Window Workspaces, Navigation & Background Splitter"
date: 2026-10-23
tags: ["lisn", "tui", "audio", "golang", "devlog", "architecture"]
---

# LISN-TUI Devlog: Multi-Window Workspaces, Navigation & Background Splitter

Today we implemented a major usability and architectural upgrade to **LISN-TUI**: a **Multi-Window Workspace System** with persistent background stem separation and bidirectional navigation.

## Why Multi-Window?

Initially, LISN-TUI was structured around sequential screen views (`ViewDrop`, `ViewSeparating`, `ViewPlayer`, etc.). While functional, once audio stem separations started taking time (or when users wanted to inspect system diagnostics, logs, or other cached songs while keeping their place in the player), the lack of window choice and easy navigation felt restrictive. Crucially, leaving the separation screen previously cancelled the separation job or stalled event listeners!

## What Changed?

1. **Dedicated Main Screen / Home Hub Opening First on Launch (`WindowHome`)**:
   - When launched interactively without CLI arguments, `LISN-TUI` now opens directly into an interactive, visual **Main Screen Dashboard**.
   - Features ASCII neon branding, live ONNX Runtime engine status, hardware accelerator diagnostics (DirectML/CUDA), and cached song counts.
   - Interactive, selectable workspace cards (`[1] ✂️ Splitter`, `[2] 🎚️ Mixer`, `[3] 📚 Library`, `[4] 🩺 Doctor`, `[5] 📜 Logs`, `[6] 💾 Export`) can be browsed with `j`/`k`/arrow keys and launched with `Enter` or direct numbers `1`-`6`.
   - Pressing `0`, `h`, or `Esc` (from standby/drop) returns to Home anytime.
   - Passing a file via CLI (e.g. `lisn song.mp3`) fast-tracks directly to the Splitter.

2. **Interactive Window Tabs in Header**:
   - The top header bar now displays all workspace windows:
     `[0] 🏠 Home │ [1] ✂️ Splitter │ [2] 🎚️ Mixer │ [3] 📚 Library │ [4] 🩺 Doctor │ [5] 📜 Logs │ [6] 💾 Export`
   - Active workspace is highlighted with a high-contrast pill, and running separation jobs show live percentage badges (`(35%)`) or running pulses.

3. **The "Splitter" as a Primary Window**:
   - The Splitter window (`WindowSplitter`) seamlessly adapts between setup mode (YouTube URL / local file input, ONNX model selection, recent files) and live execution mode (real-time progress bar, ETA, speeds, neural network logs).
   - Separation continues in the background if the user switches to another window, and pressing `F1` or `1` immediately returns to the live progress view.

4. **Instant Shortcuts to Enter Each Window**:
   - **Universal Function Keys**: `F1` (Splitter), `F2` (Mixer), `F3` (Library), `F4` (Doctor), `F5` (Logs), `F6` (Export).
   - **Direct Number Shortcuts**: `0` through `6` when outside text inputs (and outside mixer track selection).
   - **Mnemonic Keys**: `h` (Home), `s` (Splitter), `m` (Mixer), `c` (Cache/Library), `l` (Logs), `e` (Export).
   - **Command Palette**: `:home`, `:splitter`, `:split`, `:mixer`, `:stems`, `:player`, `:cache`, `:doctor`, `:logs`, `:win`.

5. **Bidirectional Back & Forth Traversal**:
   - `[` and `]` cycle between windows sequentially (outside of player looping points).
   - `Ctrl+n` and `Ctrl+p` (tmux-style) or `Ctrl+Left` / `Ctrl+Right` cycle windows anywhere.
   - `Esc` and `Backspace` pop from the `viewHistory` navigation stack to return to the previously visited window.

6. **Floating Window Manager (`w`)**:
   - Pressing `w` opens a modal window manager overlay showing all windows with icons, shortcuts, descriptions, and active status indicators.

7. **Studio Mixer Standby View**:
   - Navigating to the Studio Mixer (`[2]`) before a song is loaded now presents an inviting standby interface with clear shortcuts on how to split a new track or load an existing track from the library.

All unit and integration test suites pass 100%.
