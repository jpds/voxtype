# Building Release Binaries

## Why Docker Builds Matter

Building on modern CPUs (Zen 4, etc.) can leak AVX-512/GFNI instructions into binaries via system
libstdc++, even with RUSTFLAGS set correctly. This causes SIGILL crashes on older CPUs (Zen 3,
Haswell). Docker with Ubuntu 22.04 provides a clean toolchain without AVX-512 optimizations.

Building on hosts with newer glibc (e.g. 2.43 on CachyOS/Arch) can produce binaries that won't run
on distros with older glibc. Docker containers cap the glibc requirement at the container's version
(Ubuntu 22.04 = 2.35, Ubuntu 24.04 = 2.39). **All release binaries must be built inside Docker
containers** to ensure compatibility.

## Build Strategy

A full release requires **8 Linux binaries** (3 Whisper variants and 5 ONNX variants) plus a macOS
arm64 DMG.

**CRITICAL: Every binary must be built in Docker.** Never build release binaries directly on the
host, even for AVX-512 or MIGraphX builds that require specific hardware. Run Docker locally on the
machine with the required hardware instead.

**Whisper Binaries (3):**

| Binary | Dockerfile | Docker Context | Base Image | Max glibc |
|--------|-----------|----------------|------------|-----------|
| AVX2 | `Dockerfile.build` | Remote (pre-AVX-512) | Ubuntu 22.04 | 2.35 |
| Vulkan | `Dockerfile.vulkan` | Remote (pre-AVX-512) | Ubuntu 24.04 | 2.39 |
| AVX-512 | `Dockerfile.avx512` | Local (AVX-512 host) | Ubuntu 22.04 | 2.35 |

**ONNX Binaries (all ONNX engines: Parakeet, Moonshine, SenseVoice, Paraformer, Dolphin,
Omnilingual, Cohere):**

| Binary | Dockerfile | Docker Context | Base Image | Max glibc |
|--------|-----------|----------------|------------|-----------|
| onnx-avx2 | `Dockerfile.onnx` | Remote (pre-AVX-512) | Ubuntu 24.04 | 2.39 |
| onnx-avx512 | `Dockerfile.onnx-avx512` | Local (AVX-512 host) | Ubuntu 24.04 | 2.39 |
| onnx-cuda-12 | `Dockerfile.onnx-cuda-12` | Remote (NVIDIA GPU) | nvidia/cuda:12.6.1-cudnn-devel-ubuntu24.04 | 2.39 |
| onnx-cuda-13 | `Dockerfile.onnx-cuda-13` | Remote (NVIDIA GPU) | nvidia/cuda:13.0.3-cudnn-devel-ubuntu24.04 | 2.39 |
| onnx-migraphx | `Dockerfile.onnx-migraphx` | Local (AMD GPU host) | Ubuntu 24.04 | 2.39 |

Note: ort 2.0.0-rc.12's CUDA prebuilt is selected at build time (cu12 vs cu13) based on the
ORT_CUDA_VERSION env var or build host's CUDA install. A single binary is locked to one CUDA major
version. v0.7.0 ships both onnx-cuda-12 and onnx-cuda-13; the AUR PKGBUILD or `voxtype setup gpu
--enable` symlinks voxtype-onnx-cuda to whichever variant matches the host's runtime CUDA.

Each GPU-using ONNX binary ships with its companion shared libraries (libonnxruntime_providers_*.so)
which the EP dlopens at runtime via /proc/self/exe. scripts/package.sh installs each variant into
its own subdirectory under /usr/lib/voxtype/ (cuda-12/, cuda-13/, migraphx/) so the .so files sit
alongside the binary.

Note: ONNX binaries include bundled ONNX Runtime which contains AVX-512 instructions, but ONNX
Runtime uses runtime CPU detection and falls back gracefully on older CPUs.

## GPU Feature Flags

GPU acceleration is enabled via Cargo features:

| Feature | Backend | Use Case |
|---------|---------|----------|
| `gpu-vulkan` | Vulkan | AMD GPUs, Intel GPUs, cross-platform |
| `gpu-cuda` | CUDA | NVIDIA GPUs |
| `gpu-hipblas` | ROCm/HIP | AMD GPUs (alternative to Vulkan) |
| `gpu-metal` | Metal | macOS (not applicable for Linux builds) |

**CRITICAL: Always run `cargo clean` before building with different features.**

When switching between feature sets (e.g., CPU-only to GPU-enabled, or between different GPU
backends), stale build artifacts can cause GPU support to silently fail at runtime. The binary will
compile, have a different checksum, and appear correct, but GPU acceleration won't work.

This is especially insidious because:
- The build succeeds without errors
- The binary size and checksum differ from previous builds
- `--version` reports correctly
- But GPU detection fails silently at runtime (e.g., `use gpu = 0` instead of `use gpu = 1`)

```bash
# Build with Vulkan GPU support
cargo clean && cargo build --release --features gpu-vulkan

# Build with CUDA GPU support
cargo clean && cargo build --release --features gpu-cuda

# Build CPU-only (no GPU feature)
cargo clean && cargo build --release
```

## Remote Docker Context

A remote server with a pre-AVX-512 CPU is ideal for building binaries that must be clean of AVX-512
instructions. Configure a Docker context pointing to this server.

See `CLAUDE.local.md` for local infrastructure details (this file is gitignored).

```bash
# Switch to remote Docker context for AVX2/Vulkan builds
docker context use <your-remote-context>

# Build AVX2 and Vulkan binaries (safe, no AVX-512)
VERSION=0.4.3 docker compose -f docker-compose.build.yml up avx2 vulkan

# Switch back to local for AVX-512 build
docker context use default
```

## Full Release Build Process

**CRITICAL: Always use `--no-cache` for Docker builds and `cargo clean` for local builds.**

Stale build artifacts cause two categories of failures:

1. **Docker cache** - Without `--no-cache`, Docker may reuse layers with old version numbers. This
   caused AUR packages to ship v0.4.1 binaries labeled as v0.4.5.

2. **Cargo incremental compilation** - Without `cargo clean`, switching between feature sets (e.g.,
   CPU-only to `--features gpu-vulkan`) can produce binaries where GPU support silently fails at
   runtime. The binary compiles, has a different checksum, and reports the correct version, but GPU
   acceleration doesn't work. This is undetectable without actually testing GPU functionality.

```bash
# Set version
export VERSION=0.5.0

# 1. Build Whisper + ONNX binaries on remote server (no AVX-512 contamination)
docker context use <your-remote-context>
docker compose -f docker-compose.build.yml build --no-cache avx2 vulkan onnx-avx2
docker compose -f docker-compose.build.yml up avx2 vulkan onnx-avx2

# 2. Build ONNX CUDA on remote server (has NVIDIA GPU)
docker compose -f docker-compose.build.yml build --no-cache onnx-cuda-12 onnx-cuda-13
docker compose -f docker-compose.build.yml up onnx-cuda-12 onnx-cuda-13

# 3. Copy binaries from remote Docker containers to local
mkdir -p releases/${VERSION}
docker cp macos-release-avx2-1:/output/. releases/${VERSION}/
docker cp macos-release-vulkan-1:/output/. releases/${VERSION}/
docker cp macos-release-onnx-avx2-1:/output/. releases/${VERSION}/
docker cp macos-release-onnx-cuda-1:/output/. releases/${VERSION}/

# 4. Build AVX-512 + MIGraphX binaries locally IN DOCKER (caps glibc at container version)
docker context use <your-local-context>

# Whisper AVX-512 + ONNX AVX-512 (requires AVX-512 capable host)
docker compose -f docker-compose.build.yml --profile avx512 build --no-cache avx512 onnx-avx512
docker compose -f docker-compose.build.yml --profile avx512 up avx512 onnx-avx512

# ONNX MIGraphX (requires AMD GPU host)
docker compose -f docker-compose.build.yml build --no-cache onnx-migraphx
docker compose -f docker-compose.build.yml up onnx-migraphx

# 5. VERIFY VERSIONS before uploading (critical!)
for bin in releases/${VERSION}/voxtype-*; do
  echo -n "$(basename $bin): "; $bin --version
done

# 6. Validate glibc, instruction sets, and package
./scripts/package.sh --skip-build ${VERSION}
```

## Version Verification Checklist

**Before uploading any release, verify ALL 7 binaries report the correct version:**

```bash
# Whisper binaries (3)
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-avx2 --version
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-avx512 --version
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-vulkan --version

# ONNX binaries
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-onnx-avx2 --version
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-onnx-avx512 --version
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-onnx-cuda-12 --version
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-onnx-cuda-13 --version
releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-onnx-migraphx --version
```

If versions don't match, the Docker cache is stale. Rebuild with `--no-cache`.

## Functional Verification (GPU Builds)

**Version checks and checksums are NOT sufficient to verify GPU builds.** A binary can report the
correct version, have the expected file size, and still have non-functional GPU support due to stale
build artifacts.

For GPU-enabled binaries (Vulkan, CUDA, ROCm), verify GPU is actually detected:

```bash
# Test Vulkan build - should show "use gpu = 1" and "ggml_vulkan: Found N devices"
./voxtype-${VERSION}-linux-x86_64-vulkan daemon &
sleep 3
journalctl --user -u voxtype --since "10 seconds ago" | grep -E "(use gpu|ggml_vulkan|Found.*devices)"
# Expected: "use gpu = 1", "ggml_vulkan: Found 1 Vulkan devices"
# Bad: "use gpu = 0" or "no GPU found"

# For ONNX ROCm - should show ROCm execution provider
./voxtype-${VERSION}-linux-x86_64-onnx-rocm daemon &
sleep 3
journalctl --user -u voxtype --since "10 seconds ago" | grep -iE "(rocm|execution provider)"
```

If GPU detection fails but the binary otherwise works, the build used stale artifacts. Run `cargo
clean` and rebuild.

## Validating Binaries (AVX-512 Detection)

Use `objdump` to verify binaries don't contain forbidden instructions:

```bash
# Check for AVX-512 instructions (should be 0 for AVX2/Vulkan builds)
objdump -d releases/0.4.3/voxtype-0.4.3-linux-x86_64-avx2 | grep -c zmm
objdump -d releases/0.4.3/voxtype-0.4.3-linux-x86_64-vulkan | grep -c zmm

# Check for GFNI instructions (should be 0 for AVX2/Vulkan builds)
objdump -d releases/0.4.3/voxtype-0.4.3-linux-x86_64-avx2 | grep -cE 'vgf2p8|gf2p8'

# Verify AVX-512 build DOES have AVX-512 (should be >0)
objdump -d releases/0.4.3/voxtype-0.4.3-linux-x86_64-avx512 | grep -c zmm
```

What to look for:
- `zmm` registers = 512-bit AVX-512 registers (forbidden in AVX2/Vulkan)
- `vpternlog`, `vpermt2`, `vpblendm` = AVX-512 specific instructions
- `{1to4}`, `{1to8}`, `{1to16}` = AVX-512 broadcast syntax
- `vgf2p8`, `gf2p8` = GFNI instructions (not on Zen 3)

## Validating glibc Compatibility

**CRITICAL: All release binaries must be checked for glibc version requirements.**

Building outside Docker (directly on the host) can silently link against the host's glibc, producing
binaries that won't run on distros with older glibc. This caused the v0.6.0 incident where binaries
built on CachyOS (glibc 2.43) failed on Omarchy/Arch (glibc 2.41) with:
```
/usr/bin/voxtype: /usr/lib/libm.so.6: version `GLIBC_2.43' not found
```

```bash
# Check max glibc requirement for each binary
for bin in releases/${VERSION}/voxtype-${VERSION}-linux-x86_64-*; do
  max_glibc=$(objdump -T "$bin" 2>/dev/null | grep -oP 'GLIBC_\d+\.\d+' | sort -t. -k2 -n -u | tail -1)
  echo "$(basename $bin): $max_glibc"
done
```

**Acceptable glibc versions:**

| Binary | Base Image | Max Allowed glibc |
|--------|-----------|-------------------|
| avx2 | Ubuntu 22.04 | 2.35 |
| avx512 | Ubuntu 22.04 | 2.35 |
| vulkan | Ubuntu 24.04 | 2.39 |
| onnx-avx2 | Ubuntu 24.04 | 2.39 |
| onnx-avx512 | Ubuntu 24.04 | 2.39 |
| onnx-cuda-12 | Ubuntu 24.04 | 2.39 |
| onnx-cuda-13 | Ubuntu 24.04 | 2.39 |
| onnx-migraphx | Ubuntu 24.04 | 2.39 |

If any binary exceeds its expected glibc version, it was likely built outside Docker. Rebuild it in
the appropriate Docker container.

## ONNX Binary Instruction Leakage

**IMPORTANT: ONNX binaries also need AVX-512 instruction checks**, even when built on pre-AVX-512
hardware.

The `ort` crate downloads prebuilt ONNX Runtime binaries that may contain AVX-512 instructions
regardless of the build host's CPU. This is different from Whisper builds where the leakage comes
from system libraries.

```bash
# Check ONNX binaries for AVX-512 leakage
objdump -d voxtype-*-onnx-avx2 | grep -c zmm
# If >0, the ONNX Runtime contains AVX-512 instructions
```

**Mitigation options:**
1. **Accept fallback behavior** - ONNX Runtime will fall back to non-AVX-512 code paths at runtime
   on unsupported CPUs (may cause slight performance penalty)
2. **Build ONNX Runtime from source** - Use `ORT_STRATEGY=build` to compile ONNX Runtime with
   specific CPU flags (significantly increases build time)
3. **Use `load-dynamic` feature** - Link against system ONNX Runtime instead of bundled (requires
   users to install ONNX Runtime separately)

For now, ONNX binaries may contain AVX-512 instructions from ONNX Runtime but should still run on
pre-AVX-512 CPUs via runtime fallback. Test on target hardware to verify.

## Packaging Deb and RPM

After binaries are built and validated:

```bash
# Full build + package (builds binaries if missing)
./scripts/package.sh 0.4.3

# Package only (use existing binaries)
./scripts/package.sh --skip-build 0.4.3

# Deb only
./scripts/package.sh --deb-only --skip-build 0.4.3

# RPM only
./scripts/package.sh --rpm-only --skip-build 0.4.3
```

Packages are output to `releases/${VERSION}/`:
- `voxtype_${VERSION}-1_amd64.deb`
- `voxtype-${VERSION}-1.x86_64.rpm`

Requirements: `fpm` (gem install fpm), `rpmbuild` for RPM