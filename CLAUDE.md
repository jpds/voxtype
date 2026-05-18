# Claude Code Guidelines for Voxtype

This document helps Claude Code (and human contributors) understand the voxtype codebase, make good
architectural decisions, and submit PRs that align with project standards.

## Table of Contents

- [Project Principles](#project-principles)
- [Architecture Overview](#architecture-overview)
- [Key Design Decisions](#key-design-decisions)
- [Code Style Guide](#code-style-guide)
- [Best Practices](#best-practices)
- [Roadmap](#roadmap)
- [Git Commits](#git-commits)
- [Version Bumping](#version-bumping)
- [Building Release Binaries](#building-release-binaries)
- [AUR Packages](#aur-packages)
- [Release Notes and Website News](#release-notes-and-website-news)
- [Website](#website)
- [Development Notes](#development-notes)
- [Smoke Tests](#smoke-tests)

---

## Project Principles

These principles guide all development decisions:

1. **Dead simple user experience** - Voxtype should just work. Installation, configuration, and
   daily use should be straightforward.

2. **Backwards compatibility** - Never break existing installations. Config changes must have
   sensible defaults that preserve current behavior.

3. **Performance first** - Prioritize speed and responsiveness. On desktops this means fast
   transcription; on laptops this means battery efficiency.

4. **Excellent CLI help** - The `--help` output is documentation. Every option should be clear, with
   examples where helpful.

5. **Every option configurable everywhere** - Any setting should be configurable via CLI flag,
   environment variable, or config file.

6. **Documentation in the right places** - User-facing changes go in the user manual,
   troubleshooting guide, and configuration guide as appropriate.

---

## Architecture Overview

Voxtype is a Linux-native push-to-talk voice-to-text daemon. The architecture follows a modular,
trait-based design with async event handling.

### High-Level Flow

```
Hotkey (compositor/evdev) → Audio Capture (cpal) → Transcription (whisper-rs) → Text Processing → Output (wtype/ydotool/clipboard)
```

### Core Components

| Component | Location | Purpose |
|-----------|----------|---------|
| CLI | `src/cli.rs` | Clap command definitions, also used by `build.rs` for man pages |
| Config | `src/config.rs` | TOML parsing, defaults, icon themes (~900 lines) |
| Daemon | `src/daemon.rs` | Main event loop with `tokio::select!`, state coordination |
| State | `src/state.rs` | State machine: Idle → Recording → Transcribing → Outputting |
| CPU | `src/cpu.rs` | SIGILL handler, CPU feature detection |
| Error | `src/error.rs` | `thiserror` types with user-friendly messages |

### Module Structure

```
src/
├── hotkey/           # Keyboard input detection
│   ├── mod.rs        # HotkeyListener trait, factory
│   └── evdev_listener.rs  # Kernel-level via evdev (fallback for X11)
├── audio/            # Audio I/O
│   ├── mod.rs        # AudioCapture trait, factory
│   ├── cpal_capture.rs   # PipeWire/PulseAudio/ALSA via cpal
│   └── feedback.rs   # Audio playback for cues
├── transcribe/       # Speech-to-text
│   ├── mod.rs        # Transcriber trait, factory, prepare() optimization
│   ├── whisper.rs    # Local in-process via whisper-rs
│   ├── remote.rs     # HTTP API (OpenAI-compatible)
│   ├── subprocess.rs # GPU isolation wrapper
│   └── worker.rs     # Child process entry point
├── output/           # Text delivery
│   ├── mod.rs        # TextOutput trait, factory, fallback chain
│   ├── wtype.rs      # Wayland-native (best Unicode support)
│   ├── dotool.rs     # Keyboard layout support via uinput
│   ├── ydotool.rs    # X11/TTY fallback (requires daemon)
│   ├── clipboard.rs  # Universal fallback via wl-copy
│   ├── paste.rs      # Clipboard + Ctrl+V
│   └── post_process.rs   # LLM cleanup command
├── text/             # Text transformations
│   └── mod.rs        # Spoken punctuation, replacements
└── setup/            # Installation helpers
    ├── model.rs      # Model selection & download
    ├── gpu.rs        # GPU feature detection
    ├── waybar.rs     # Waybar config snippets
    ├── systemd.rs    # Service installation
    └── compositor.rs # Hyprland/Sway/River keybinding setup
```

### Trait-Based Extensibility

Each major component defines a trait allowing multiple implementations:

| Trait | Implementations | Extension Point |
|-------|----------------|-----------------|
| `HotkeyListener` | `EvdevListener` | Add libinput, compositor-specific listeners |
| `AudioCapture` | `CpalCapture` | Add JACK, direct ALSA support |
| `Transcriber` | `WhisperTranscriber`, `RemoteTranscriber`, `SubprocessTranscriber` | Add new ASR backends |
| `TextOutput` | `WtypeOutput`, `DotoolOutput`, `YdotoolOutput`, `ClipboardOutput` | Add X11, compositor-specific output |

---

## Key Design Decisions

Understanding why things are built a certain way helps you extend them correctly.

### Async Runtime (Tokio)

**Why:** Push-to-talk requires responsive hotkey detection while handling long I/O operations.

**Pattern:**
- Main loop uses `tokio::select!` to multiplex hotkey events, signals, and task completion
- Audio capture uses mpsc channels to stream data without blocking
- Transcription runs via `spawn_blocking` to avoid blocking the event loop
- Model loading is a background task hidden behind recording time

### Hotkey Detection

**Preferred:** Compositor keybindings (Hyprland, Sway, River) - native integration, no special
permissions needed. Voxtype provides `voxtype record start/stop/toggle` commands for compositor
bindings to call.

**Fallback:** evdev listener - works on X11 and as a universal fallback. Requires user to be in
`input` group.

Set `[hotkey] enabled = false` when using compositor keybindings.

### GPU Memory and Performance

**Priority:** Performance is critical. Fast transcription on desktops, battery efficiency on
laptops.

**Trade-off:** GPU memory isn't released after in-process transcription, which causes memory growth
over time. The `gpu_isolation = true` option spawns a child process that exits after transcription,
releasing GPU memory.

**Guidance:** Don't assume users want GPU isolation by default. Some users prioritize keeping the
model loaded for faster subsequent transcriptions. Let users choose based on their hardware and
usage patterns.

### Output Fallback Chain

**Why:** No single output method works everywhere (wtype needs Wayland, ydotool needs daemon, dotool
needs uinput access).

**Chain:** wtype → dotool → ydotool → clipboard

- **wtype**: Wayland-native, best Unicode/CJK support, no daemon needed
- **dotool**: Works on X11/Wayland/TTY, supports keyboard layouts via `DOTOOL_XKB_LAYOUT`, no daemon
  needed
- **ydotool**: Works on X11/Wayland/TTY, requires ydotoold daemon
- **clipboard**: Universal fallback via wl-copy

Each method is probed before use; failures cascade to next method.

### Configuration Layering

**Priority (highest wins):**
1. CLI arguments
2. Environment variables (`VOXTYPE_*`)
3. Config file (`~/.config/voxtype/config.toml`)
4. Built-in defaults

This allows overriding any setting at any level without modifying config files.

### CPU Compatibility via SIGILL Handler

**Why:** Binaries built on modern CPUs can contain instructions that crash on older CPUs.

**Solution:** Install SIGILL handler via `.init_array` constructor (runs before `main()`). If
triggered, displays helpful message instead of silent crash.

---

## Code Style Guide

### Rust Conventions

- Run `cargo fmt` before committing
- Run `cargo clippy -- -D warnings` and fix all warnings
- Use `cargo test` to verify changes

### Naming

| Item | Convention | Example |
|------|-----------|---------|
| Modules | snake_case | `audio_capture`, `post_process` |
| Types/Structs | PascalCase | `AudioCapture`, `TextProcessor` |
| Functions/Methods | snake_case | `create_transcriber`, `start_recording` |
| Config fields | snake_case in TOML | `on_demand_loading`, `max_duration_secs` |

### Error Handling

Use `thiserror` with user-friendly messages that include remediation steps:

```rust
#[error("Cannot open input device '{0}'. Is the user in the 'input' group?\n  Run: sudo usermod -aG input $USER")]
DeviceAccess(String),
```

Group related errors into domain-specific types:
- `VoxtypeError` - top-level
- `HotkeyError` - with group/key setup instructions
- `AudioError` - with device listing hints
- `TranscribeError` - with model download suggestions
- `OutputError` - with setup instructions for each method

### Logging

Use `tracing` (not `log`):

```rust
use tracing::{info, debug, warn, error};

info!("Starting daemon");
debug!(device = %device_name, "Opening audio device");
warn!("Model not found, downloading...");
error!(?err, "Transcription failed");
```

Worker processes log to stderr only (stdout reserved for IPC).

### Module Organization

- Keep trait definitions in `mod.rs`
- Put implementations in separate files
- Factory functions go in `mod.rs`
- Tests go at the bottom of each file in a `#[cfg(test)]` module

### Comments

- Prefer self-documenting code over comments
- Add comments for non-obvious "why" decisions
- Use `///` doc comments for public APIs
- Avoid TODO comments; open issues instead

---

## Best Practices

### Backwards Compatibility

**This is critical.** Never break existing installations.

- New config fields must have defaults that preserve current behavior
- Removed fields should be silently ignored, not cause errors
- CLI changes must not break existing scripts or keybindings
- Test upgrades by running the new version with an old config file

### Adding a New Transcription Backend

1. Create `src/transcribe/your_backend.rs`
2. Implement the `Transcriber` trait
3. Add variant to the factory in `src/transcribe/mod.rs`
4. Add configuration fields to `src/config.rs` with sensible defaults
5. Add CLI flags in `src/cli.rs` with clear `--help` text
6. Document in `docs/CONFIGURATION.md`
7. Add tests

### Adding a New Output Method

1. Create `src/output/your_method.rs`
2. Implement the `TextOutput` trait
3. Add to fallback chain in `src/output/mod.rs` if appropriate
4. Consider whether it should be a fallback or explicit selection

### Modifying Configuration

- Add new fields with sensible defaults (backward compatible)
- Update `src/config.rs` default values
- Add corresponding CLI flags in `src/cli.rs`
- Update `docs/CONFIGURATION.md`
- If the field affects behavior significantly, mention in release notes

### Documentation Requirements

When adding user-facing features, update:
- `docs/USER_MANUAL.md` - How to use the feature
- `docs/CONFIGURATION.md` - Config file options
- `docs/TROUBLESHOOTING.md` - If there are failure modes users might hit
- CLI `--help` text - Via clap attributes in `src/cli.rs`

### Testing Changes

```bash
# Run all tests
cargo test

# Run specific test
cargo test test_name

# Run with output visible
cargo test -- --nocapture

# Test a specific module
cargo test text::

# Manual testing
cargo run -- -vv  # Verbose daemon
cargo run -- transcribe test.wav  # Test transcription
cargo run -- status --follow  # Watch state changes
```

### Performance Considerations

- Avoid allocations in the hot path (hotkey detection, audio streaming)
- Use `spawn_blocking` for CPU-intensive work
- The `prepare()` method on `Transcriber` allows hiding model load time behind recording time
- Prefer streaming over buffering where possible
- On laptops, battery efficiency matters as much as raw speed

### Avoid Over-Engineering

- Don't add abstraction layers until there are multiple implementations
- Don't add configuration for edge cases; handle them with sensible defaults
- Three similar lines of code are better than a premature abstraction
- Only validate at system boundaries (user input, external APIs)

---

## Roadmap

See [docs/claude/ROADMAP.md](docs/claude/ROADMAP.md) for packaging priorities, the feature roadmap,
blocked items, and non-goals.

---

## Git Commits

- **Whenever discussing submitting work or creating a PR, remind the user to target `dev`, not
  `main`.** The `dev` branch is the integration branch; `main` tracks stable releases.
- **NEVER commit without GPG signing.** All commits must be signed. Do not use `--no-gpg-sign` or
  skip signing for any reason.
- **Pull requests with unsigned commits will be rejected.** Every commit in a PR must be signed.
- If GPG signing fails, stop and inform the user rather than bypassing signing.

### Crediting Contributors

When work builds on contributions from others, always include appropriate credit:

- **Use `Co-authored-by:` trailers** for commits that incorporate someone else's work, even if
  substantially modified
- **When in doubt, give credit.** It's better to over-attribute than to omit someone's contribution
- **Credit applies broadly:** code, ideas, bug reports, design feedback, and review comments all
  warrant acknowledgment
- **Check PR and issue history** to identify contributors whose work influenced the commit

Examples of when to add co-author credit:
- Cherry-picking or rebasing commits from a PR (even if you resolve conflicts or make changes)
- Implementing a feature based on someone's detailed issue or design proposal
- Fixing a bug that someone else identified and diagnosed
- Incorporating code snippets or approaches suggested in review comments

Format:
```
Co-authored-by: Name <email@example.com>
```

Multiple co-authors are fine when several people contributed to the work.

## Version Bumping

**When bumping the version in Cargo.toml, ALWAYS update Cargo.lock before committing.**

The AUR source package (`voxtype`) uses `cargo fetch --locked` and `cargo build --frozen`, which
require Cargo.lock to exactly match Cargo.toml. If the version in Cargo.lock doesn't match
Cargo.toml, the build fails.

```bash
# Correct version bump process:
# 1. Edit Cargo.toml to set new version
# 2. Run cargo build to update Cargo.lock
cargo build
# 3. Verify Cargo.lock was updated
grep -A2 'name = "voxtype"' Cargo.lock  # Should show new version
# 4. Commit BOTH files together
git add Cargo.toml Cargo.lock
git commit -S -m "Bump version to X.Y.Z"
```

**Never commit a version bump to Cargo.toml without also committing the updated Cargo.lock.**

This caused the v0.4.6 incident where users building from source got:
```
error: the lock file Cargo.lock needs to be updated but --locked was passed to prevent this
```

## Building Release Binaries

Release engineering procedures (Docker build matrix, GPU feature flags,
version/glibc/instruction-set validation, deb/rpm packaging) live in
[docs/claude/RELEASE_BUILDS.md](docs/claude/RELEASE_BUILDS.md). The `/build-release`,
`/validate-binaries`, and `/package-release` skills consult it.

## AUR Packages

AUR publishing rules (channel split between `voxtype-bin` and `voxtype-bin-rc`, `pkgver` vs `pkgrel`
policy, post-upgrade message) live in [docs/claude/AUR.md](docs/claude/AUR.md). The `/aur-publish`
skill consults it.

## Release Notes and Website News

Style guide and checklist for GitHub release notes and the matching website news article live in
[docs/claude/RELEASE_NOTES.md](docs/claude/RELEASE_NOTES.md). The `/update-docs` skill consults it.

## Website

The website at voxtype.io is hosted via GitHub Pages. It deploys automatically when changes to
`website/` are merged to main. No separate deployment step is needed.

## Development Notes

### Killing the Daemon

When using `pkill voxtype` or manually killing the daemon, Waybar status followers (`voxtype status
--follow`) will also be terminated. After restarting the daemon:

```bash
# Either reload Waybar entirely
pkill -SIGUSR2 waybar

# Or the followers will reconnect on next Waybar restart
```

The systemd unit restart (`systemctl --user restart voxtype`) handles this gracefully, but manual
kills require Waybar attention.

### Binary Location Priority

The PATH typically has `~/.local/bin` before `/usr/local/bin`. When testing new builds:

```bash
# Check which binary is active
which voxtype

# Remove stale local copy if needed
rm ~/.local/bin/voxtype
hash -r  # Clear shell's command cache
```

## Smoke Tests

See [docs/SMOKE_TESTS.md](docs/SMOKE_TESTS.md) for comprehensive manual testing procedures.

For automated regression testing, use the `/regression-test` skill which covers unit tests, CLI
commands, config validation, and binary variant verification.