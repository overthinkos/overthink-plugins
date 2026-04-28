---
name: build-toolchain
description: |
  C/C++ build toolchain with gcc, cmake, autoconf, ninja, git, and pkg-config.
  Use when working with native compilation, build tools, or C/C++ development.
---

# build-toolchain -- C/C++ compilation toolchain

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CCACHE_DISABLE` | `1` |

## Packages (41 total, grouped by purpose)

The package list is the **centralized place** to add system libraries that any builder
stage needs. Image-specific cargo deps live here when they're broadly useful; truly
image-specific deps stay in their image's own layer (e.g., `niri` still ships its own
clang-devel etc. for niri-specific builds).

### 1. Generic C/C++ build tools (21)

`autoconf`, `automake`, `binutils`, `bison`, `ccache`, `cli11-devel`, `cmake`, `flex`,
`gcc`, `gcc-c++`, `gdb`, `git`, `glibc-devel`, `libtool`, `make`, `ninja-build`, `patch`,
`pkgconf`, `pkgconf-m4`, `pkgconf-pkg-config`, `redhat-rpm-config`, `rpm-build`, `rpm-sign`.

### 2. Rust toolchain (2)

`rust`, `cargo`. Used by `setuptools-rust` and any layer that compiles a Smithay-based
Wayland compositor from source. See `/ov-layers:rust` for the disambiguation between
this builder-stage Rust and the runtime `rust` layer.

### 3. Bindgen runtime + assembler (2)

`clang-devel` — provides `libclang.so`, required by bindgen-based crates such as
`x264-sys` and `ffmpeg-sys-next`. Without it, those crates panic in their `build.rs` at
`bindgen-0.72.1/lib.rs:616:27`.

`nasm` — assembler for SIMD-accelerated native deps. The `turbojpeg-sys` crate vendors
libjpeg-turbo, whose CMake build calls `enable_language(ASM_NASM)` and fails without
nasm with `No CMAKE_ASM_NASM_COMPILER could be found`.

### 4. Smithay backend dev headers (10)

`wayland-devel`, `libudev-devel`, `libxkbcommon-devel`, `mesa-libgbm-devel`,
`libseat-devel`, `libinput-devel`, `systemd-devel`, `libdrm-devel`, `pixman-devel`,
`libjpeg-turbo-devel`.

These satisfy smithay's default feature set when pixelflux_wayland builds against
`smithay = { rev = "ca932e04…" }` with features
`["backend_drm", "backend_egl", "backend_gbm", "backend_libinput", "backend_udev",
"renderer_gl", "renderer_pixman", "use_system_lib", "desktop", "wayland_frontend"]`.
Each `backend_*` and `renderer_*` feature pulls a pkg-config lookup against the matching
`-devel` package at link time.

### 5. Codec dev libs from RPM Fusion free (3)

`libva-devel`, `x264-devel`, `ffmpeg-devel`.

These live in **RPM Fusion free** (not in the base Fedora repos), which is why
`/ov-images:fedora-builder` now includes `/ov-layers:rpmfusion` **before**
`build-toolchain` in its layer list. They are required for compiling pixelflux's
`libva-sys`, `x264-sys`, and `ffmpeg-sys-next` cargo crates against the system
codec libraries.

See `/ov-layers:selkies` (Patched pixelflux build pipeline) for the consumer story —
the selkies layer's `build.sh` clones pixelflux from a pinned commit, applies four
inline source patches, and runs `pip install .` against this builder stage.

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch), `deb:` (Debian/Ubuntu) — full parity. The deb section maps Fedora's ~40 `-devel` packages to their Debian `-dev` equivalents (`libclang-dev`, `libavformat-dev`, `libdrm-dev`, `libinput-dev`, `libturbojpeg0-dev`, `libseat-dev`, `libva-dev`, `libwayland-dev`, `libxkbcommon-dev`, `libgbm-dev`, `libsystemd-dev`, `libpixman-1-dev`, `libx264-dev`, `libudev-dev` — plus `build-essential`). Drops on deb: `redhat-rpm-config`, `rpm-build`, `rpm-sign`, `pkgconf-m4` (RPM-specific); `cli11-devel` (header-only, cmake FetchContent).

## Usage

```yaml
# image.yml
my-image:
  layers:
    - build-toolchain
```

## Used In Images

- `/ov-images:fedora-builder`
- `/ov-images:archlinux-builder`
- `/ov-images:fedora-remote` (via remote layer ref)
- Also used in `bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:pixi` — Sibling in builder images for conda-forge package builds
- `/ov-layers:nodejs` — Sibling in builder images for npm package builds
- `/ov-layers:yay` — Pairs in archlinux-builder for AUR builds
- `/ov-layers:rust` — Disambiguates the builder-stage `rust`+`cargo` packages here from the runtime `rust` layer
- `/ov-layers:rpmfusion` — Must be applied **before** this layer in fedora-builder so codec dev libs (`x264-devel`, `ffmpeg-devel`) can install
- `/ov-layers:selkies` — Primary consumer of all the new cargo/codec deps (Patched pixelflux build pipeline)

## Related Commands
- `/ov:build` — Multi-stage builders consume this layer for pixi/npm/cargo builds
- `/ov:generate` — See how builder stages are generated from this layer

## When to Use This Skill

Use when the user asks about:

- C/C++ compilation inside containers
- Build tools (cmake, make, ninja, autoconf)
- Compiler toolchain setup
- The `build-toolchain` layer or its packages

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
