# Karaoke (Demucs) Build Pipeline

This repository builds the binaries and model that the Muuzbox Karaoke
feature downloads at user activation. The Electron app itself lives in a
separate repository — this repo only produces and publishes the runtime
artifacts.

Two GitHub Actions workflows:

| Workflow | What it builds | Frequency |
|---|---|---|
| [build-demucs-binary.yml](../.github/workflows/build-demucs-binary.yml) | `demucs-cli` for win-x64 / mac-arm64 / linux-x64 | Whenever upstream demucs.cpp updates, or we want a clean rebuild |
| [build-demucs-model.yml](../.github/workflows/build-demucs-model.yml) | `htdemucs.bin` (ggml-converted Demucs weights) — same file all platforms | Rarely — only when Demucs releases a new model |

Both workflows are **manual** (`workflow_dispatch`). Triggered from the
GitHub Actions UI; no scheduled runs.

## End-to-end procedure for a release

1. **Pick a `bundle_version`** — e.g. `demucs-cli-v0.1.0`. This tag is shared
   between the two workflows and must match `BUNDLE_VERSION` in the
   Electron app's `src/services/karaokeInstaller.js`.
2. **Run `Build Demucs.cpp CLI`** with that version. It produces three
   binaries (win-x64, mac-arm64, linux-x64) and a manifest, and creates
   / updates the matching GitHub Release on this repo.
3. **Run `Build Demucs.cpp Model`** with the SAME version. It converts
   PyTorch weights to ggml, attaches the `.bin` to the existing release,
   and amends the release body with the model's SHA-256.
4. **Copy the SHA-256s from the release body** into the Electron app's
   `karaokeInstaller.js` (`KARAOKE_BUNDLE.bin.sha256` and `model.sha256`).
   Bump `BUNDLE_VERSION` if it changed.
5. **Verify the URLs in `KARAOKE_BUNDLE`** point to the new release assets
   (or to the Hetzner mirror — see "Mirroring to Hetzner" below).
6. Commit the Electron-side change and ship the next Electron build.

## URL pattern the Electron installer expects

The Electron app pulls these exact URLs (set in `karaokeInstaller.js`):

```
https://github.com/ZookeeMedia/muuzbox-karaoke-build/releases/download/<bundle_version>/demucs-cli-<platform>[.exe]
https://github.com/ZookeeMedia/muuzbox-karaoke-build/releases/download/<bundle_version>/htdemucs.bin
```

The GitHub Actions release step (in build-demucs-binary.yml) names files
exactly that way, so the URLs work out of the box. If you mirror to
Hetzner the URLs need to point at `cdn.muuzbox.com/karaoke/...` instead.

## Mirroring binaries to Hetzner CDN

GitHub Releases are fine as the source of truth, but mirroring to Hetzner
gets us:

- A predictable URL pattern not tied to a third party
- Bandwidth not metered against GitHub's quotas
- Consistent CDN performance from venue locations (Dubai, Thailand)

To mirror after a release:

```bash
TAG=demucs-cli-v0.1.0
gh release download "$TAG" --repo ZookeeMedia/muuzbox-karaoke-build --dir /tmp/karaoke-$TAG

# Push to Hetzner. Adjust the remote path to match your hosting layout.
rsync -avz --progress /tmp/karaoke-$TAG/ \
  user@cdn.muuzbox.com:/srv/cdn/karaoke/$TAG/
```

The installer downloads exactly:

```
https://cdn.muuzbox.com/karaoke/<bundle_version>/<platform>/demucs-cli[.exe]
https://cdn.muuzbox.com/karaoke/<bundle_version>/htdemucs.bin
```

so the upload structure should match.

## Code signing

The workflows look for these GitHub secrets to sign binaries; if any are
missing, signing is skipped and the binary ships unsigned (fine for
dev / internal test, fails Gatekeeper / SmartScreen in production).

| Secret | Used by |
|---|---|
| `WINDOWS_CERT_BASE64` | Windows signtool (base64-encoded `.pfx`) |
| `WINDOWS_CERT_PASSWORD` | Windows signtool |
| `MACOS_CERT_BASE64` | macOS codesign (base64-encoded `.p12`) |
| `MACOS_CERT_PASSWORD` | macOS codesign |
| `MACOS_SIGNING_IDENTITY` | e.g. `Developer ID Application: Your Name (TEAMID)` |

**Apple notarisation is NOT done in this workflow.** Binary will sign
cleanly but `xcrun notarytool submit` against Apple's service is the
next step we'd add. For now, expect first-launch Gatekeeper prompts on
macOS until we wire that in.

## Known unknowns (validate on first run)

The workflows were written against documented behaviour of demucs.cpp
but have not been run end-to-end. The first dispatch will likely
surface a small adjustment or two:

1. **Binary name produced by CMake** — the workflow searches for
   `demucs.cpp.main` under `build/`. If upstream renames it, update
   `matrix.bin_name` in `build-demucs-binary.yml`. The "Locate binary"
   step prints the `build/` tree on failure to help diagnose.

2. **Conversion script location** — `model-conversion` step tries
   `scripts/convert-pth-to-ggml.py` then `convert.py`. If neither
   exists, inspect the upstream repo and adjust.

3. **PyTorch wheel availability** — the workflow pins PyTorch via the
   CPU wheel index. If that index changes, the install step might fail.
   Falling back to the default index is fine for the conversion
   workflow (it'll just download GPU wheels we don't need).

4. **libsndfile linkage on Windows** — vcpkg static linkage is what we
   use, matching how demucs.cpp expects it. If linkage fails, the
   alternative is the dynamic triplet (`x64-windows`) plus shipping the
   DLL alongside the .exe.

5. ~~**macOS x64 runners are deprecated by GitHub**~~ — resolved:
   `mac-x64` was dropped from the matrix in 2026-06. The macos-13 runner
   pool was retired; venue Mac installs are all Apple Silicon. If Intel
   support is needed in future, cross-compile from arm64 with
   `-arch x86_64` and `lipo`, or re-add the matrix entry when GitHub
   ships a replacement Intel runner.

## Updating to a new demucs.cpp version

1. Note the new upstream tag or SHA.
2. Run `Build Demucs.cpp CLI`, set `demucs_cpp_ref` to the new ref and
   bump `bundle_version` (e.g. `v0.1.0` → `v0.2.0`).
3. If the model changed too (rare), run the model workflow with the
   same version.
4. Update `KARAOKE_BUNDLE` in the Electron app's `karaokeInstaller.js`.
5. Ship the next Electron build.

## Cost / time budget

Per binary release (3 platforms):

* Wall-clock: ~15 min (parallel matrix builds — bound by the slowest, usually Windows)
* GitHub Actions minutes consumed: ~17 (Windows counts 2× and macOS arm64 counts 10× against the quota, so it's more than wall-clock suggests)

Per model release: ~25 min wall-clock, ~25 minutes consumed.

These run rarely; well within free / paid tier limits for any reasonable
Electron-app cadence.
