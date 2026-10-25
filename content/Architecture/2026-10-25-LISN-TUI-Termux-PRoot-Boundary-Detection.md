---
title: "LISN-TUI Architecture: Termux Host vs. PRoot Container Boundary Detection"
date: 2026-10-25
tags: ["lisn-tui", "architecture", "termux", "proot", "container-detection", "android"]
---

# LISN-TUI Architecture: Termux Host vs. PRoot Container Boundary Detection

This document specifies the mechanism used by the universal Bash runner (`run.sh`) to detect and differentiate between the Android Termux host environment and the internal PRoot Linux container.

---

## 1. The Container Re-installation & Boundary Bug

When executing `run.sh` in Termux, two related issues emerged:
1. **Container Re-installation Failure**: `proot-distro list` formatting varied across Termux versions; a naive `grep -A 2 "ubuntu"` failed to match `Status: installed`, causing `run.sh` to execute `proot-distro install ubuntu` on an already existing container. Under `set -e`, this terminated execution with:
   ```
   Error: container 'ubuntu' already exists. Specify a different name with '--name NAME'.
   ```
2. **Environment Variable Inheritance**: When entering PRoot Ubuntu via `proot-distro login ubuntu`, the child shell inherited the parent's `$TERMUX_VERSION` and access to `/data/data/com.termux`. Without a strict differentiator, nested scripts inside PRoot incorrectly identified themselves as the outer Android host.

---

## 2. Invariant: Package Manager Boundary Differentiator

The definitive boundary marker between Termux host and PRoot Ubuntu is the native Termux wrapper `pkg`:
- **In Termux Host**: `command -v pkg` evaluates to `true` (Termux ships with `pkg`).
- **Inside PRoot Ubuntu**: `command -v pkg` evaluates to `false` (standard Debian/Ubuntu uses `apt`, never `pkg`).

### Implementation Contract:
```bash
if command -v pkg >/dev/null 2>&1 && ([ -n "$TERMUX_VERSION" ] || [ -d "/data/data/com.termux" ]); then
    # Running strictly on outer Termux host
fi
```

---

## 3. Filesystem-Level Container Validation

Instead of parsing ambiguous multi-line CLI text output from `proot-distro list`, `run.sh` verifies container installation directly against the rootfs directory structure:
```bash
UBUNTU_INSTALLED=false
if [ -d "$PREFIX/var/lib/proot-distro/installed-rootfs/ubuntu" ] || [ -d "/data/data/com.termux/files/usr/var/lib/proot-distro/installed-rootfs/ubuntu" ]; then
    UBUNTU_INSTALLED=true
elif proot-distro login ubuntu -- true 2>/dev/null; then
    UBUNTU_INSTALLED=true
fi

if [ "$UBUNTU_INSTALLED" = false ]; then
    proot-distro install ubuntu || true
fi
```
This guarantees idempotent, safe execution regardless of whether Ubuntu is already installed or fresh.
