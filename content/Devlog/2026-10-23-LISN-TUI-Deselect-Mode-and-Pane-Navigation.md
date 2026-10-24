---
title: "LISN-TUI Devlog: Deselect Mode, Multi-Pane Navigation, and Instant Cached Playback"
date: 2026-10-23
tags: ["tui", "ux", "bubbletea", "keyboard-shortcuts", "lisn-tui", "modal-navigation"]
---

# LISN-TUI Devlog: Deselect Mode, Multi-Pane Navigation, and Instant Cached Playback

Today we implemented explicit **Selected** vs. **Not Selected** pane states controlled by `ESC` in LISN-TUI's Splitter view.

## What Was Improved

1. **Pane Focus State via `ESC`**:
   - Panes can now exist in two distinct states: **`Selected`** or **`Not Selected`** (Deselected).
   - When any pane is selected (e.g. Pane 1 Audio Track or Pane 2 Recent Separations), pressing `ESC` deselects it, entering the **`○ NO PANE SELECTED`** state.
   - Both panes return to standard borders without glowing highlights.

2. **Pressing `ESC` to Go Back**:
   - When in the **`No Pane Selected`** state, pressing `ESC` brings you back to the Home screen (`m.goBack()`), preserving the exact back functionality.

3. **Selecting Panes from Deselected State**:
   - Pressing `1`, `Enter`, or `i` selects Pane 1 and focuses the text input.
   - Pressing `2`, `Tab`, or `j` / `↓` selects Pane 2 (Recent Cached Separations).
   - Pressing `ESC` when a pane is active immediately deselects it.

4. **Instant Zero-Latency Playback**:
   - When Pane 2 is selected, pressing `Enter` on any cached separation immediately starts playback in the 8-stem Studio Mixer (`ViewPlayer`).
   - Pressing `d` opens the Acoustic Detective report.
   - Pressing `x` initiates deletion confirmation.

All automated unit tests (`go test -v ./pkg/tui/...` and `go test ./...`) pass cleanly.
