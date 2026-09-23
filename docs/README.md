# Browser Virtual Desktop

A full-featured, original x86 PC emulator that runs entirely in a browser tab, deployed as a static site on GitHub Pages. No plugins, no installs, no server for the core system — boot real Linux or Windows inside a webpage.

This is a solo project, built from scratch (QEMU-inspired architecture, but completely original — no borrowed code), under real constraints: a locked-down/managed Chromebook with no Linux, no Developer Mode, no DevTools, and no installs. Implementation happens by hand in [github.dev](https://github.dev) and/or GitHub Codespaces, with AI assistance where useful.

## What it does

- Emulates a full x86 PC — CPU, MMU, firmware, and devices — in JavaScript
- Boots real, unmodified Linux and Windows
- Supports **x86_64** and **x86_32** (as one mode-aware CPU core), with a generalized architecture framework for adding others (e.g. ARM) later
- Supports both **legacy BIOS** and **UEFI** firmware, selectable per VM, with dynamically-generated ACPI tables matching each VM's actual configuration
- Full **multi-core** support — multiple virtual CPUs coordinating over shared memory, with correct x86 memory-ordering semantics (LOCK-prefixed instructions mapped to `Atomics`)
- Multiple VMs at once: separate tabs run independent VMs, and a single tab can hold several switchable VM configs
- Currently offline-only; real guest-OS networking is a planned future addition (see [Networking](#networking))

## How it works, roughly

```
Browser tab
 └── VM Manager (tracks configs, switches between them)
      └── VM Instance
           ├── N virtual CPU cores (Web Workers + SharedArrayBuffer RAM)
           ├── MMU (paging + segmentation, x86-faithful, bounded/LRU TLB)
           ├── Firmware (real 16-bit-executed BIOS, or UEFI)
           ├── Devices (disk, keyboard/mouse, timer, video)
           └── Virtual disk (chunked storage in IndexedDB)
```

The CPU core executes real x86 machine code — including the firmware itself, which runs as genuine 16-bit real-mode code on the emulated CPU, the same way a real BIOS chip would. From there, an unmodified Linux kernel or Windows bootloader takes over exactly as it would on real hardware.

OS images (kernels, firmware, disk images) are pre-built by GitHub Actions ahead of time and published to this repo's `gh-pages` branch — you never wait for a build; you just pick an OS from what's already available.

## Status

**Architecture planning: complete, including resolution of an external feasibility review.** Implementation is in progress by hand, one piece at a time, starting with the CPU decoder's smallest working loop.

| Subsystem | Status |
|---|---|
| CPU abstraction framework + interpreter strategy | Planned |
| x86_64 / x86_32 core | Planned |
| MMU (paging, segmentation, bounded/LRU TLB) | Planned |
| Firmware (legacy BIOS + UEFI, ACPI, CPUID) | Planned |
| FPU/SIMD (x87, SSE, AVX) | Planned |
| OS boot paths (Linux, Windows) | Planned |
| Device model + multi-core (incl. TSO/Atomics) | Planned |
| Multi-instance (tabs/configs) | Planned |
| Storage (chunked IndexedDB) | Planned |
| Build/deployment pipeline | Planned |
| Networking (Render-hosted proxy) | Directional — not yet detailed |

See `full-scope-plan.txt` in this repo for the complete, detailed design doc, including a section-by-section summary of how each item from an external feasibility review was resolved.

## Performance

A naive JS interpreter is far too slow to boot something like Windows in reasonable time. The plan uses a tiered approach:

1. **A disciplined, WASM-portable interpreter** — dispatch-table opcode lookup, typed-array memory access, no per-instruction allocation — built first, for correctness
2. **A WASM-compiled hot loop**, once the interpreter is measured (not assumed) to be too slow
3. **JIT / dynamic binary translation**, only if tier 2 genuinely isn't enough

## Networking

Guest OSes currently have **no real network access** — this isn't a bug, it's a hard browser limitation: no web page (including WASM, Workers, or anything else running in a browser tab) can open raw TCP/UDP sockets, by design, on any browser. Anything that only needs the virtual disk — offline games, local tools, sideloaded software — works fine without it.

Real networking is planned as a **future addition**, via a small backend hosted on [Render](https://render.com) (or similar) acting as a WebSocket-to-TCP proxy: the guest's virtual NIC sends packets over a WebSocket to that backend, which makes real connections on the guest's behalf. This requires maintaining a small piece of always-on infrastructure separate from the static site, and detailed design of that piece hasn't happened yet.

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

*This project runs cross-origin isolated via a service-worker header injection ([`coi-serviceworker`](https://github.com/gzuidhof/coi-serviceworker)) to enable `SharedArrayBuffer`, since GitHub Pages can't set the required headers natively. First-time visitors will see one automatic reload as this is set up.*