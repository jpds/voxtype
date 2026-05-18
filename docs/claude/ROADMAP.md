# Roadmap

## Packaging Priority

Expanding distribution support is a current focus:

1. **NixOS** - Next priority for packaging
2. **Manjaro** - Sway/Hyprland ecosystem support
3. **Other Sway/Hyprland distros** - Expand reach to tiling WM users
4. **Homebrew on Linux** ([#177](https://github.com/peteonrails/voxtype/issues/177))
5. **Silverblue / atomic distros** ([#178](https://github.com/peteonrails/voxtype/issues/178))

Existing packages: Arch (AUR: `voxtype`, `voxtype-bin`), Debian (.deb), Fedora (.rpm)

## Feature Roadmap

Based on open issues and project direction.

**v0.7.1 (confirmed):**
- **voxtype-models CDN** - Host every ONNX engine model voxtype downloads (Cohere variants,
  Parakeet, Moonshine, SenseVoice, Paraformer, Dolphin, Omnilingual, ECAPA-TDNN diarization) on
  Cloudflare R2 behind `models.voxtype.io`. Removes the dependency on community HF accounts
  (`csukuangfj/*`, `istupakov/*`, `onnx-community/*`). Plumb a `models_base_url` indirection in
  `src/setup/model.rs`, ship per-model `manifest.json` with sha256s, validate downloads against the
  manifest, and write a mirror script that pulls upstream HF and pushes to R2 byte-identically. HF
  stays as a fallback so users behind firewalls keep working.
- **Streaming transcription** ([#283](https://github.com/peteonrails/voxtype/issues/283)) -
  Parakeet-first, English-first push-to-stream mode. On a parallel agent's branch.

**Near Term:**
- **Deterministic integration tests** - Automated smoke tests using pre-recorded audio files that
  can run in CI without LLM/human interaction
- **Meeting echo cancellation edge trimming** - Remove residual bleed-through words at segment
  boundaries when loopback audio is active. GTCRN handles the bulk of echo removal, but 1-2 stray
  words can appear at the start/end of mic segments where the STFT window crosses a chunk boundary.

**Medium Term:**
- **Audio caching** ([#28](https://github.com/peteonrails/voxtype/issues/28)) - Save recordings for
  replay/re-transcription
- **Audio + output history** ([#209](https://github.com/peteonrails/voxtype/issues/209)) - Companion
  to audio caching; surface past dictations
- **Native StatusNotifierItem tray** ([#267](https://github.com/peteonrails/voxtype/issues/267)) -
  Replace XEmbed tray for KDE Plasma / GNOME compatibility
- **OpenAI-compatible local STT API** ([#244](https://github.com/peteonrails/voxtype/issues/244)) -
  Single daemon serves hotkey dictation + HTTP API for other tools

**Exploratory:**
- **Consolidated release binaries** - Reduce from 8 binaries today (avx2, avx512, vulkan, onnx-avx2,
  onnx-avx512, onnx-cuda-12, onnx-cuda-13, onnx-migraphx) to 3 (cpu, cuda, migraphx) by combining
  Whisper + Vulkan + ONNX engines into each binary. Vulkan and CUDA/MIGraphX fall back to CPU when
  no GPU is present, and ONNX Runtime does runtime CPU dispatch. Trade-off is losing AVX-512 Whisper
  performance (~10-30%) and larger binaries. Blocked on whisper.cpp/ggml adding runtime SIMD
  dispatch if AVX-512 performance must be preserved; otherwise, AVX2-only Whisper is safe on all
  x86-64 CPUs.
- **Nemotron Speech backend** ([#47](https://github.com/peteonrails/voxtype/issues/47)) -
  Alternative ASR engine
- **Vibe Voice backend** ([#285](https://github.com/peteonrails/voxtype/issues/285)) - Microsoft's
  speech model
- **Dictation Intents** ([#231](https://github.com/peteonrails/voxtype/issues/231)) - Configurable
  per-shortcut behavior (translate vs transcribe, custom prompts)
- **Parakeet sortformer for meeting diarization** - Evaluate parakeet-rs's sortformer feature as
  alternative to the current ml-diarization ECAPA-TDNN pipeline

**Blocked/Waiting:**
- **Nixpkgs onnxruntime MIGraphX support** - Verify the nixpkgs `onnxruntime` build (with
  `rocmSupport = true`) actually exposes the MIGraphX EP. The Nix flake's `parakeet-migraphx` output
  uses `onnxruntimeRocm` and sets `ORT_MIGRAPHX_MODEL_CACHE_PATH`; if MIGraphX isn't exposed in
  nixpkgs, ORT will fail to register the EP at runtime.
- **Cohere decoder on CUDA** - Encoder runs on GPU; decoder pinned to CPU pending ORT's CUDA
  `GroupQueryAttention` kernel adding `attention_bias` support. Flip the second arg of
  `build_session(&decoder_file, threads, "decoder", false)` in `src/transcribe/cohere.rs` once ORT
  lands the kernel.

## Non-Goals

- Windows/macOS support (Linux-first, Wayland-native)
- GUI configuration (GTK/Qt/web). A TUI (`voxtype configure`) is supported and surfaced as a
  desktop-file launcher entry; CLI and config file remain the primary interfaces for scripting and
  headless setups.
- Continuous dictation mode (push-to-talk is the paradigm)