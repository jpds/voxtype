# AUR Packages

- AUR repos are nested git repos in `packaging/arch/`, `packaging/arch-bin/`, and
  `packaging/arch-bin-rc/`
- These directories are ignored by the main repo (in `.gitignore`)
- To publish to AUR: `cd packaging/arch && git add -A && git commit -m "message" && git push`
- GPG signing key for AUR repos: `E79F5BAF8CD51A806AA27DBB7DA2709247D75BC6`

## Two AUR channels: voxtype-bin vs voxtype-bin-rc

There are two prebuilt-binary AUR packages:

- **`voxtype-bin`** tracks the latest GitHub *stable* release. This is what 99% of users want.
- **`voxtype-bin-rc`** tracks the latest GitHub *pre-release* tag (e.g., `v0.7.3-rc1`). For users
  who want to test RC builds without giving up their stable install path.

The split exists because users running prebuilt binaries shouldn't accidentally upgrade onto an RC
and inherit its rough edges. By keeping them as separate AUR packages, the stable channel only
advances when stable does.

**conflicts/provides setup.** `voxtype-bin-rc` declares `provides=('voxtype')` and
`conflicts=('voxtype' 'voxtype-bin')`. Stable `voxtype-bin` keeps its existing
`conflicts=('voxtype')`. So a user can only have one channel installed at a time, and tools that
depend on `voxtype` (e.g., `voxtype-osd`) are satisfied by either package. Switching is a
remove-then-install: `yay -R voxtype-bin && yay -S voxtype-bin-rc`.

**Maintenance burden.** Each RC tag requires the same update workflow on `packaging/arch-bin-rc/` as
a stable release requires on `packaging/arch-bin/`:

1. Bump `pkgver` (using the dot-separated form, e.g., `0.7.3.rc1`)
2. Update `sha256sums` from the uploaded GitHub pre-release artifacts
3. Regenerate `.SRCINFO` via `makepkg --printsrcinfo > .SRCINFO`
4. Commit and push the nested git repo to `aur:voxtype-bin-rc`

That's twice the AUR work per cycle. The `aur-publish` skill should be extended to cover both
channels.

**The `_upstream_ver` trick.** AUR forbids hyphens in `pkgver` because the hyphen delimits `pkgrel`
(`pkgver-pkgrel`). GitHub release tags for RCs use hyphens (`v0.7.3-rc1`). The `voxtype-bin-rc`
PKGBUILD stores the version as `pkgver=0.7.3.rc1` and computes `_upstream_ver="${pkgver/.rc/-rc}"`
to derive the real tag (`0.7.3-rc1`) for download URLs. If we ever change the RC naming convention
upstream (e.g., to `v0.7.3-beta1`), the mangling rule in `_upstream_ver` has to change too.

## AUR Versioning: pkgver vs pkgrel

**For the `voxtype-bin` package, always bump `pkgver`, never just `pkgrel` when binaries change.**

The binary download URLs include `pkgver` but not `pkgrel`:
```
https://github.com/peteonrails/voxtype/releases/download/v$pkgver/voxtype-$pkgver-linux-x86_64-avx2
```

When only `pkgrel` is bumped, the URL stays the same. AUR helpers like yay cache PKGBUILDs and see
"same URL = same file," causing checksum failures when binaries have actually changed.

**When to use each:**

| Scenario | Action |
|----------|--------|
| New binary release | Bump `pkgver`, reset `pkgrel` to 1, create new GitHub release |
| Fix PKGBUILD only (deps, install script) | Bump `pkgrel` |
| Binaries were wrong/corrupted | **Release new version** (bump `pkgver`), don't try to fix in place |

**Never do this:**
- Re-upload different binaries to an existing GitHub release
- Bump only `pkgrel` when binary content has changed

This caused the v0.4.5 incident where users had cached PKGBUILDs with old checksums that didn't
match re-uploaded binaries.

## Post-Install Message

When updating the AUR packages, also update the post-upgrade message in
`packaging/arch-bin/voxtype-bin.install` to reflect the current release highlights.

The `post_upgrade()` function displays a message to users after they upgrade. This should summarize
what's new in the version they just installed, not old releases.

```bash
# Check current message
cat packaging/arch-bin/voxtype-bin.install

# Update the post_upgrade() message with current version highlights
# Then commit with the PKGBUILD changes
```