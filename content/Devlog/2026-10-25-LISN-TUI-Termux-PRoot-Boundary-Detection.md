---
title: "Devlog: Fixing Termux Container Existence Check and PRoot Boundary"
date: 2026-10-25
tags: ["lisn-tui", "devlog", "termux", "bugfix", "proot", "android"]
---

# Devlog: Fixing Termux Container Existence Check and PRoot Boundary

We resolved an edge-case where `run.sh` attempted to reinstall an existing Ubuntu container in Termux when the user already had Ubuntu installed in `proot-distro`.

---

## 1. Root Cause
- The previous check relied on `grep -A 2` output from `proot-distro list`. On devices where `proot-distro` prints extra alias or description fields, the pattern failed to match `installed`, triggering `proot-distro install ubuntu` on an existing container.
- Under `set -e`, the exit code 1 from `proot-distro` aborted the script before it could enter the container.

---

## 2. The Solution
1. Switched to filesystem directory detection (`$PREFIX/var/lib/proot-distro/installed-rootfs/ubuntu`) paired with a fallback test (`proot-distro login ubuntu -- true`).
2. Added `|| true` to the installer call to ensure robustness.
3. Differentiated Termux host from PRoot container using `command -v pkg`, which exists exclusively on the host.

Users can re-run the universal one-liner in Termux without conflicts.
