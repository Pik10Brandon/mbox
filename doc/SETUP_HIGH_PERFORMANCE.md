# High-performance setup and run guide

This guide walks through building **mbox** with an optimized configuration and running it with strong sandbox coverage. It keeps the defaults simple while ensuring the sandbox uses available kernel features (seccomp/BPF and ptrace) and that binaries are built with release-grade flags.

## Prerequisites

1. Install build tools and headers (example for Debian/Ubuntu):
   ```bash
   sudo apt update
   sudo apt install -y build-essential pkg-config libcap-dev
   ```
2. From the repository root, work inside `src/`:
   ```bash
   cd src
   ```

## Build steps (optimized)

1. Copy the default configuration template:
   ```bash
   cp {.,}configsbox.h
   ```
2. Configure with release-friendly flags (keeps debug symbols off and enables common optimizations):
   ```bash
   CFLAGS="-O2 -pipe" ./configure --prefix=/usr/local
   ```
3. Compile using all available cores:
   ```bash
   make -j"$(nproc)"
   ```
4. Run the built-in checks to verify syscall interception logic:
   ```bash
   ./testall.sh
   ```

## Running with maximum protections

* To enable seccomp/BPF filtering (when supported by the kernel), use `-s`:
  ```bash
  ./mbox -s -- /bin/echo "hello from mbox"
  ```
* To cut network access entirely, pair seccomp with isolation flags:
  ```bash
  ./mbox -s -n -i -- /bin/echo "offline sandbox"
  ```
* For profile-based policies (filesystem/network allowlists or blocks), craft a profile under `doc/NOTE.profile` format and run:
  ```bash
  ./mbox -p /path/to/profile -s -- <command>
  ```

## Installation (optional)

After testing, install the binary and manpage into `/usr/local`:
```bash
sudo make install
```

## Quick health checklist

- `./mbox -h` succeeds and lists options (including `-s` for seccomp/BPF).
- `./testall.sh` passes on your target kernel.
- The sandboxed commands run with `-s` do not require elevated privileges.

These steps give you a repeatable, optimized build plus a runnable configuration that exercises mbox's strongest available sandboxing modes.
