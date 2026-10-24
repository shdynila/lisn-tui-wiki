---
title: "LISN-TUI Devlog: Standardizing ESC Navigation Across All Windows"
date: 2026-10-24
tags: ["tui", "ux", "bubbletea", "keyboard-navigation", "lisn-tui", "devlog"]
---

# LISN-TUI Devlog: Standardizing ESC Navigation Across All Windows

Today we unified the `ESC` key behavior across every workspace window in **LISN-TUI**, replicating the two-tier deselect and return-to-home model originally designed for the Splitter view.

## The Two-Tier ESC Pattern

Navigating terminal UIs should feel intuitive, predictable, and forgiving. Previously, pressing `ESC` had disparate meanings depending on which window was open:
- In the **Splitter**, pressing `ESC` unselected the active pane, and a second press returned to Home.
- In the **Studio Mixer**, pressing `ESC` while playing audio did nothing at all.
- In **Library**, **Doctor**, **Logs**, and **Export**, pressing `ESC` instantly exited the window, making it impossible to unfocus an element without leaving the screen.

Now, **all 7 windows** adhere to a single, unified rule:
1. **1st ESC Press**: **Unselects the active pane / enters Deselect Mode**.
   - The selection cursor (`▶ `) disappears.
   - Text input cursors blur.
   - Active neon glowing border styles soften to standard card borders.
   - Channel controls in the studio mixer are safely guarded against index panics.
2. **2nd ESC Press**: **Returns directly to the Home Dashboard (`WindowHome`)**.
   - If already in Deselect Mode, `ESC` returns you straight to the main landing screen.
   - On the Home screen itself, the first `ESC` deselects the workspace choice card, and a second `ESC` safely remains on Home.

## Re-Selecting from Deselect Mode

Re-engaging focus from Deselect Mode requires zero friction:
- **Studio Mixer**: Pressing `↑/↓`, `k/j`, or channel numbers `1-8` immediately selects that channel.
- **Library**: Pressing `↑/↓`, `k/j`, or `Enter` selects the song row.
- **Splitter**: Pressing `1` selects audio input; pressing `2` selects recent separations.
- **Export**: Pressing `i` or `Enter` focuses the destination path input.
- **Doctor & Logs**: Pressing `Tab`, `Enter`, or arrow keys re-focuses the diagnostics or log stream.

## Automated Verification

We added `TestUniversalEscDeselectAndHome` to `pkg/tui/app_test.go`, verifying the two-step ESC sequence and re-selection triggers across all windows:
- `ViewHome`
- `ViewDrop` (Splitter)
- `ViewPlayer` (Studio Mixer)
- `ViewCache` (Library)
- `ViewDoctor` (Diagnostics)
- `ViewLogs` (Log Telemetry)
- `ViewExport` (Mixdown Exporter)

All automated tests (`go test ./...`) pass cleanly, and the production binary builds without warnings.
