---
title: "Devlog: Running Neural Stem Separation in Your Pocket with Termux on Android"
date: 2026-10-25
tags: ["lisn-tui", "devlog", "android", "termux", "mobile", "arm64", "pulseaudio", "cross-platform"]
---

# Devlog: Running Neural Stem Separation in Your Pocket with Termux on Android

Today we expanded LISN-TUI beyond desktop workstations (Windows, macOS, Linux x86_64) into the mobile domain: running full neural audio stem separation directly on Android smartphones using **Termux**.

---

## The Vision: Offline Neural Audio Workstation in Termux

Termux is a staple tool for developers who demand command-line sovereignty on Android. With LISN-TUI, a smartphone transforms into a standalone stem isolation and studio mixing console:
- Download tracks from YouTube or local storage
- Separate vocals, drums, bass, and synthesizers in real-time
- Solo/mute stems in a multi-track TUI mixer
- Listen directly through phone speakers, USB-C headphones, or Bluetooth audio

---

## Engineering Hurdles & Solutions

### 1. The Glibc vs. Bionic Barrier
`yalue/onnxruntime_go` uses CGO to bind to Microsoft's official ONNX Runtime shared library. However:
- Termux binaries link against Android's native Bionic libc.
- Microsoft only publishes official Linux binaries linked against GNU glibc.

By deploying via `proot-distro install ubuntu`, we get a complete glibc ARM64 userspace on the phone. This allows LISN-TUI to download and run official `libonnxruntime.so` binaries with zero compatibility hacks.

### 2. Audio Routing via PulseAudio TCP Sink
Android restricts `/dev/snd` to prevent rogue background apps from hijacking the audio hardware. To enable playback from PRoot:
1. In Termux, start PulseAudio binding to `127.0.0.1` with OpenSL ES audio output.
2. In PRoot Ubuntu, export `PULSE_SERVER=127.0.0.1`.
All Beep/Oto streams now flow seamlessly to the phone speakers.

### 3. Automated Decompression of Linux Tarballs
Unlike Windows `.zip` releases, official Linux ARM64 releases are `.tgz` archives. We wrote `extractLibFromTgz` in `pkg/engine/onnx.go` using pure Go (`archive/tar` + `compress/gzip`). The binary automatically downloads, unpacks, and links `libonnxruntime.so` on first run without requiring any manual file operations.

---

## Step-by-Step Mobile Setup Guide

For any user looking to run LISN-TUI on their phone:

```bash
# 1. In Termux (install proot-distro and pulseaudio)
pkg update && pkg install -y proot-distro pulseaudio

# 2. Start PulseAudio audio server
pulseaudio --start --load="module-native-protocol-tcp auth-ip-acl=127.0.0.1 auth-anonymous=1" --exit-idle-time=-1

# 3. Enter Ubuntu environment
proot-distro install ubuntu
proot-distro login ubuntu

# 4. Install build dependencies inside Ubuntu
apt update && apt install -y git golang ffmpeg yt-dlp pulseaudio build-essential
export PULSE_SERVER=127.0.0.1

# 5. Clone and build LISN-TUI
git clone https://github.com/shdynila/lisn-tui.git
cd lisn-tui
go build -o bin/lisn ./cmd/lisn

# 6. Launch!
./bin/lisn
```

---

## Verification
- Added in-memory unit tests (`TestExtractLibFromTgz`) covering tar extraction and symlink fallback.
- Validated on `feat/android-arm64-linux-support` and merged into `main` via PR #6.
