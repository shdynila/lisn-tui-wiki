---
title: "LISN-TUI Architecture: Direct Non-Recursive Termux Bootstrap & TTY Reattachment"
date: 2026-10-25
tags: ["lisn-tui", "architecture", "termux", "proot", "tty", "unix-pipes", "bugfix"]
---

# LISN-TUI Architecture: Direct Non-Recursive Termux Bootstrap & TTY Reattachment

This document specifies the direct, non-recursive container bootstrap and terminal I/O reattachment model for LISN-TUI on Android Termux.

---

## 1. The Recursive Execution Loop Analysis

When `run.sh` logged into PRoot Ubuntu, it previously invoked `curl ... run.sh | bash` inside the container. Because PRoot inherits the host's environment variables (including `$TERMUX_VERSION` and `/data/data/com.termux`), the child script inside Ubuntu incorrectly identified itself as running on the outer Android host.

This caused the child process to call `proot-distro` from within PRoot, leading to:
```
Warning: proot-distro should not be executed as root user...
Error: container 'ubuntu' is busy.
proot error: execve("/usr/bin/bash"): No such file or directory
```

---

## 2. Invariant: Strict Host/Container Boundary & Non-Recursive Flow

1. **Host Boundary Check**:
   ```bash
   if [ -z "$IN_PROOT" ] && [ ! -f "/etc/debian_version" ] && [ ! -f "/etc/os-release" ] && ([ -n "$TERMUX_VERSION" ] || [ -d "/data/data/com.termux" ]); then
   ```
   Inside Debian/Ubuntu, `/etc/debian_version` is guaranteed to exist, permanently suppressing the outer Termux host branch.

2. **Direct Inline Bootstrap**:
   Instead of recursing into `run.sh`, the outer host script executes a direct inline provisioning block inside PRoot:
   - Verifies media libraries (`ffmpeg`, `yt-dlp`, `pulseaudio`).
   - Fetches the pre-compiled `lisn_linux_arm64.tar.gz` directly into `/usr/local/bin/lisn`.
   - Executes `lisn` with `$@`.

---

## 3. Interactive Terminal Reattachment (`/dev/tty`)

When users invoke an installer via `curl -sSL ... | bash`, standard input (`fd 0`) is connected to the network pipe rather than the user's interactive terminal. This prevents terminal UIs from receiving keyboard input and causes PRoot path sanitization failures:
```
proot warning: can't sanitize binding "/proc/self/fd/0": No such file or directory
```

To restore interactive terminal capability, `run.sh` reattaches `fd 0` immediately upon start:
```bash
if [ ! -t 0 ] && [ -e /dev/tty ]; then
    exec < /dev/tty
fi
```
This guarantees full keyboard event handling for the Bubbletea TUI application.
