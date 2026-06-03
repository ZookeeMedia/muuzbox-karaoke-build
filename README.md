# muuzbox-karaoke-build

Build pipeline that produces the [demucs.cpp](https://github.com/sevagh/demucs.cpp)
binaries and Demucs model used by the Karaoke feature of the Muuzbox
Electron app.

The Electron app itself lives in a separate repository on Bitbucket;
this repo is intentionally narrow — it exists only to produce signed,
verified runtime artifacts and host them on GitHub Releases.

## Quick start

1. Open the **Actions** tab on GitHub.
2. Run **Build Demucs.cpp CLI** with a `bundle_version` like
   `demucs-cli-v0.1.0`. Wait ~15 min for the matrix to finish.
3. Run **Build Demucs.cpp Model** with the same `bundle_version`. Wait
   ~25 min.
4. Open the matching release on the **Releases** page — copy the binary
   and model SHA-256s from the release body into `KARAOKE_BUNDLE` in the
   Electron app's `src/services/karaokeInstaller.js`.
5. (Optional) Mirror to Hetzner per the operational doc.

Full procedure, code-signing setup, known unknowns, and Hetzner mirror
instructions live in [docs/karaoke-build-pipeline.md](docs/karaoke-build-pipeline.md).

## What the Electron app expects

After a successful release, these URLs serve the artifacts directly:

```
https://github.com/ZookeeMedia/muuzbox-karaoke-build/releases/download/<tag>/demucs-cli-win-x64.exe
https://github.com/ZookeeMedia/muuzbox-karaoke-build/releases/download/<tag>/demucs-cli-mac-arm64
https://github.com/ZookeeMedia/muuzbox-karaoke-build/releases/download/<tag>/demucs-cli-mac-x64
https://github.com/ZookeeMedia/muuzbox-karaoke-build/releases/download/<tag>/demucs-cli-linux-x64
https://github.com/ZookeeMedia/muuzbox-karaoke-build/releases/download/<tag>/htdemucs.bin
```

The installer verifies each file by SHA-256 before activating.

## Upstream tracking

* [sevagh/demucs.cpp](https://github.com/sevagh/demucs.cpp) — the C++
  port we build from. The model conversion script lives there too.
* [facebookresearch/demucs](https://github.com/facebookresearch/demucs) — 
  source of the PyTorch weights we convert.

## License notes

The build artifacts produced here are licensed under demucs.cpp's
license (MIT). The Demucs model weights themselves are MIT-licensed by
Meta. Bundling and redistributing both in the Muuzbox installer is fine.
