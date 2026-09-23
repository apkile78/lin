# lin

# Browser Virtual Desktop

A full-featured, original x86 PC emulator that runs entirely in a browser tab, deployed as a static site on GitHub Pages. No plugins, no installs, no server — boot real Linux or Windows inside a webpage.

This is a solo project, built from scratch (QEMU-inspired architecture, but completely original — no borrowed code), under real constraints: a locked-down/managed Chromebook with no Linux, no Developer Mode, no DevTools, no installs, and no AI-assisted coding tools. Everything is written by hand in [github.dev](https://github.dev) (press `.` on this repo, or swap `github.com` → `github.dev` in the URL).

## What it does

- Emulates a full x86 PC — CPU, MMU, firmware, and devices — in JavaScript
- Boots real, unmodified Linux and Windows
- Supports **x86_64** and **x86_32** (as one mode-aware CPU core), with a generalized architecture framework for adding others (e.g. ARM) later
- Supports both **legacy BIOS** and **UEFI** firmware, selectable per VM
- Full **multi-core** support — multiple virtual CPUs coordinating over shared memory, not just single-core
- Multiple VMs at once: separate tabs run independent VMs, and a single tab can hold several switchable VM configs
- Runs offline — no real network access from the guest OS (see [Scope](#scope--whats-intentionally-not-here))

## How it works, roughly

```
Browser tab
 └── VM Manager (tracks configs, switches between them)
      └── VM Instance
           ├── N virtual CPU cores (Web Workers + SharedArrayBuffer RAM)
           ├── MMU (paging + segmentation, x86-faithful)
           ├── Firmware (real 16-bit-executed BIOS, or UEFI)
           ├── Devices (disk, keyboard/mouse, timer, video)
           └── Virtual disk (chunked storage in IndexedDB)
```

The CPU core executes real x86 machine code — including the firmware itself, which runs as genuine 16-bit real-mode code on the emulated CPU, the same way a real BIOS chip would. From there, an unmodified Linux kernel or Windows bootloader takes over exactly as it would on real hardware.

OS images (kernels, firmware, disk images) are pre-built by GitHub Actions ahead of time and published to this repo's `gh-pages` branch — you never wait for a build; you just pick an OS from what's already available.

## Status

**Architecture planning: complete.** Every subsystem below has been fully designed — implementation is in progress by hand, one piece at a time, starting with the CPU decoder's smallest working loop.

| Subsystem | Status |
|---|---|
| CPU abstraction framework | Planned |
| x86_64 / x86_32 core | Planned |
| MMU (paging, segmentation, TLB) | Planned |
| Firmware (legacy BIOS + UEFI) | Planned |
| FPU/SIMD (x87, SSE, AVX) | Planned |
| OS boot paths (Linux, Windows) | Planned |
| Device model + multi-core | Planned |
| Multi-instance (tabs/configs) | Planned |
| Storage (chunked IndexedDB) | Planned |
| Build/deployment pipeline | Planned |

See `full-scope-plan.txt` in this repo for the complete, detailed design doc covering every decision above.

## Scope — what's intentionally *not* here

- **No real networking.** Guest OSes can't reach the actual internet — this is a hard browser sandboxing limit (no raw sockets are exposed to any web page, ever, by any browser), not a bug. Anything that only needs the virtual disk — offline games, local tools, sideloaded software — works fine. Anything needing a live connection (multiplayer, package managers, license checks) won't.
- **No AI-assisted or Codespaces-based development.** Every line is written by hand.

## Repo structure

```
/src                                  - VM source (JavaScript)
/kernel-configs                       - Per-distro Buildroot configs
/index.html                           - Entry point
/.github/workflows/build-kernel.yml   - Manually-triggered, per-target build workflow

gh-pages branch:
/kernels/<distro>/vmlinuz             - Pre-built kernels, one per distro
/available-images.json                - Manifest of what's ready to use
/index.html, /src/                    - Served to users
```

## Building an OS image

Builds are **manual and targeted** — triggering a build only rebuilds the specific OS/architecture/firmware combination you select, never the whole matrix. Go to the **Actions** tab → run `build-kernel.yml` → pick your target(s).

## Running it

Open the published GitHub Pages site, pick an OS and firmware type from the setup screen, and boot.

---

*This project runs cross-origin isolated via a service-worker header injection ([`coi-serviceworker`](https://github.com/gzuidhof/coi-serviceworker)) to enable `SharedArrayBuffer`, since GitHub Pages can't set the required headers natively.*