---
title: "Devlog: Consolidating Installers to Universal Bash and PowerShell"
date: 2026-10-25
tags: ["lisn-tui", "devlog", "bash", "powershell", "ux", "simplification", "termux"]
---

# Devlog: Consolidating Installers to Universal Bash and PowerShell

Today we streamlined the client deployment workflow by eliminating separate platform-specific bootstrap scripts and standardizing on just two universal entrypoints: **Bash** (`run.sh`) and **PowerShell** (`run.ps1`).

---

## 1. What Changed

Previously, Android users ran `termux.sh`, while Linux/macOS users ran `run.sh`. 

We merged the complete Termux automated bootstrapping flow into `run.sh`:
- Automatic `$TERMUX_VERSION` and directory detection.
- Headless PulseAudio audio bridge startup.
- PRoot Ubuntu container validation and setup.
- Symlink and wrapper installation (`$PREFIX/bin/lisn`).

Now, whether a user is on an Android smartphone in Termux, an Ubuntu workstation, or a MacBook, the command is identical:

```bash
curl -sSL https://raw.githubusercontent.com/shdynila/lisn-tui-releases/main/run.sh | bash
```

And for Windows users:
```powershell
irm https://raw.githubusercontent.com/shdynila/lisn-tui-releases/main/run.ps1 | iex
```

---

## 2. Updated Portal & Website

The GitHub Pages download portal at [https://shdynila.github.io/lisn-tui-releases/](https://shdynila.github.io/lisn-tui-releases/) has been updated:
- The 4 platform tabs were replaced with 2 sleek, high-visibility options: **Bash** and **PowerShell**.
- Automatic browser detection selects PowerShell on Windows and Bash on Linux/macOS/Android.
- Removed redundant `termux.sh` artifact.
