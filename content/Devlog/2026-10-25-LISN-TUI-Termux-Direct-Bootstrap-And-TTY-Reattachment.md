---
title: "Devlog: Eliminating Recursive PRoot Calls and Fixing TTY Pipes in Termux"
date: 2026-10-25
tags: ["lisn-tui", "devlog", "termux", "bugfix", "proot", "tty", "android"]
---

# Devlog: Eliminating Recursive PRoot Calls and Fixing TTY Pipes in Termux

We diagnosed and resolved the root cause of the `container 'ubuntu' is busy` and `/proc/self/fd/0` warnings encountered when running the universal installer in Termux.

---

## 1. What Happened
1. When `run.sh` entered Ubuntu, it invoked `curl ... run.sh | bash` inside the container.
2. Because environment variables were inherited, the script inside Ubuntu thought it was still on the Android host and attempted to invoke `proot-distro` again.
3. Because standard input was connected to a pipe from `curl`, PRoot failed to sanitize `/proc/self/fd/0`.

---

## 2. The Fix
1. **Reattached `/dev/tty`**: The script reattaches `stdin` to `/dev/tty` at the very beginning, restoring interactive keyboard input and resolving PRoot descriptor warnings.
2. **Eliminated Nested Calls**: Replaced the nested `curl ... run.sh` call inside Ubuntu with direct, self-contained installation and execution of `lisn`.
3. **Strict Boundary Checks**: Added checks for `/etc/debian_version` and `/etc/os-release` to guarantee that code inside the PRoot container never attempts host operations.
