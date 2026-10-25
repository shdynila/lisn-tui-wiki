---
title: "Devlog: Launching Public Distribution Hub, GitHub Pages Portal, and Termux Bootstrap"
date: 2026-10-25
tags: ["lisn-tui", "devlog", "releases", "termux", "github-pages", "ci-cd", "distribution"]
---

# Devlog: Launching Public Distribution Hub, GitHub Pages Portal, and Termux Bootstrap

Following the successful implementation of ARM64 Linux and universal ESC navigation, we implemented an automated release and distribution system modeled after the `0_ath_client` $\rightarrow$ `0_ath_releases` pipeline.

---

## 1. The Need for Public Distribution

LISN-TUI's core development repository is private. Previously, running the application on a new machine or Android phone required setting up GitHub CLI authentication, Personal Access Tokens, or manual file transfers from a PC.

By standing up **[shdynila/lisn-tui-releases](https://github.com/shdynila/lisn-tui-releases)**, we established a public release hub that:
1. Keeps the source code private.
2. Delivers pre-compiled, standalone binaries to any user via simple one-line scripts.
3. Hosts a modern, glassmorphic Hugo download portal on GitHub Pages.

---

## 2. Zero-Config One-Line Installers

Users can now install and run LISN-TUI anywhere with a single terminal command:

### 📱 Android Phone (Termux)
```bash
curl -sSL https://raw.githubusercontent.com/shdynila/lisn-tui-releases/main/termux.sh | bash
```
The script automatically configures PulseAudio TCP audio routing to Android OpenSL ES, sets up an Ubuntu PRoot environment, downloads the pre-built ARM64 binary, and drops a persistent `lisn` command into Termux.

### 🐧 Linux & 🍎 macOS
```bash
curl -sSL https://raw.githubusercontent.com/shdynila/lisn-tui-releases/main/run.sh | bash
```

### 🪟 Windows PowerShell
```powershell
irm https://raw.githubusercontent.com/shdynila/lisn-tui-releases/main/run.ps1 | iex
```

---

## 3. Automated CI/CD Release Matrix

We configured `.github/workflows/release.yml` in `lisn-tui` to build and upload release assets across 5 distinct targets:
- Linux x86_64 (`lisn_linux_amd64.tar.gz`)
- Linux ARM64 (`lisn_linux_arm64.tar.gz`)
- Windows x86_64 (`lisn_windows_amd64.zip`)
- macOS Apple Silicon (`lisn_mac_silicon.tar.gz`)
- macOS Intel (`lisn_mac_intel.tar.gz`)

Assets are uploaded automatically to the public release repository using the repository secret `PUBLIC_REPO_PAT`.

---

## 4. Verification & Status
- **Public Repository**: [https://github.com/shdynila/lisn-tui-releases](https://github.com/shdynila/lisn-tui-releases)
- **GitHub Pages Portal**: [https://shdynila.github.io/lisn-tui-releases/](https://shdynila.github.io/lisn-tui-releases/)
- **Release Matrix PR**: [shdynila/lisn-tui#8](https://github.com/shdynila/lisn-tui/pull/8) (merged)
