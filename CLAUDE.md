# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the Linux kernel (v6.19, "Baby Opossum Posse"), licensed under GPL-2.0-only. It uses the Kbuild recursive build system with Kconfig for configuration.

## Build Commands

```bash
# Configure
make defconfig                    # Default config for current arch
make menuconfig                   # Interactive ncurses config

# Build
make                              # Build kernel + modules
make vmlinux                      # Kernel image only
make modules                      # Loadable modules only
make drivers/gpu/drm/             # Build all files in a directory
make drivers/gpu/drm/i915/i915.ko # Build a single module

# Install
make modules_install              # Install modules
make headers_install              # Install UAPI headers

# Clean
make clean                        # Remove generated files, keep config
make mrproper                     # Remove all generated files + config

# Useful flags
V=1                               # Verbose build output
O=dir                             # Out-of-tree build output
W=1                               # Extra compiler warnings
C=1                               # Run sparse on re-compiled files
```

## Style and Linting

- **8-character tabs** for indentation (not spaces) in C code; 80-column line limit
- K&R brace style: opening brace on same line for statements, next line for function definitions
- Use `/* */` comments only (no `//` in C code)
- `switch`/`case` at same indentation level

```bash
scripts/checkpatch.pl --file <file>    # Style-check a file
scripts/checkpatch.pl <patch>          # Style-check a patch
make C=1                                # Sparse static analysis
make coccicheck                         # Coccinelle semantic patches
```

`.clang-format` and `.editorconfig` are provided for editor integration.

## Testing

```bash
make kselftest                    # Run kernel selftests (requires installed kernel)
make kselftest-all                # Build selftests only
tools/testing/kunit/kunit.py run  # KUnit unit tests (runs in UML/QEMU)
```

Runtime sanitizers (enabled via Kconfig): KASAN, KMSAN, KCSAN, KFENCE, lockdep, kmemleak.

## Architecture

Build order defined in top-level `Kbuild`: `init/ -> usr/ -> arch/ -> kernel/ -> certs/ -> mm/ -> fs/ -> ipc/ -> security/ -> crypto/ -> io_uring/ -> rust/ -> drivers/ -> sound/ -> net/ -> virt/`

Key subsystem directories:
- `arch/` — Architecture-specific code (x86, arm64, riscv, etc.)
- `kernel/` — Core: scheduler (`sched/`), syscalls, signals, cgroups, BPF, tracing, RCU, IRQ, workqueues
- `mm/` — Memory management: allocators, virtual memory, page cache, huge pages, OOM killer
- `fs/` — Filesystems (ext4, btrfs, xfs, tmpfs, procfs, sysfs, etc.)
- `net/` — Networking stack (TCP/UDP, IPv4/IPv6, netfilter, wireless, Bluetooth, XDP)
- `drivers/` — Device drivers (largest directory; GPU/drm, net/ethernet, sound/alsa, USB, PCI, etc.)
- `block/` — Block I/O layer (blk-mq, I/O schedulers)
- `security/` — LSMs: SELinux, AppArmor, Landlock, integrity/IMA/EVM
- `init/main.c` — `start_kernel()`: primary kernel boot entry point

Configuration flows: `Kconfig` (top-level) -> `arch/*/Kconfig` -> subsystem `Kconfig` files -> `.config`

## AI Contribution Requirements

Per `Documentation/process/coding-assistants.rst`:
- All code must be GPL-2.0-only compatible with SPDX identifiers
- **Never add `Signed-off-by` tags** — only humans can certify the DCO
- Use `Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]` for attribution
- Follow standard kernel development process and coding style

## Key References

- Finding maintainers: `scripts/get_maintainer.pl <file>`
- Development process: `Documentation/process/development-process.rst`
- Submitting patches: `Documentation/process/submitting-patches.rst`
- Coding style: `Documentation/process/coding-style.rst`
- Build system docs: `Documentation/kbuild/`
- Driver API: `Documentation/driver-api/`
- Build documentation: `make htmldocs`
