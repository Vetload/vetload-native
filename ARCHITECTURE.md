# Architecture: Vetload native stack (C03)

Builds the native tools and libraries Vetload's engine runs as child processes, from pinned, checksummed sources, published as images with digests, SBOM and provenance. Brief: `../vetload-platform/docs/components/C03-native-stack.md` (private); cross-component conventions: [ADR-0049](https://github.com/Vetload/vetload-platform/blob/main/docs/architecture/decisions/0049-wave-1-alignment.md).

## Modules

| Path | Purpose |
| --- | --- |
| `sources.json` | Source manifest |
| `recipes/<name>.sh` | Build script per entry, exact flags |
| `Dockerfile` | Stages `fetch`, `build`, `base`, `sdk` |
| `launcher/vl-spawn/` | Child launcher: limits and process group before `exec` |
| `wrappers/vl-pdfium/` | PDFium structure dump and page render, in C |
| `guard/` | Checksums, licence guard, component audit, sizes, reproducibility |
| `sbom/`, `schemas/` | CycloneDX; JSON Schemas for the formats below |
| `.github/workflows/` | Pull-request builds, tag releases, scheduled controls |

## Source manifest: `sources.json`

Top level: `schema` (1), `bases` (images by digest, [ADR-0002](docs/architecture/decisions/0002-base-operating-system-image.md)) and `sources`:

| Field | Meaning |
| --- | --- |
| `name`, `version` | Unique SBOM name; upstream version |
| `url` | HTTPS release tarball, never a generated git archive |
| `mirror` | Our release-asset copy (P0) |
| `sha256`, `sha256_origin` | Enforced checksum; `upstream-file`, `github-asset-digest` or `computed-after-signature` |
| `signature` | Optional `{url, key}`, checked when the pin changes |
| `license` | SPDX expression |
| `role`, `linkage` | `runtime`, `build-tool`, `prebuilt`; `shared`, `executable`, `build-only` (LGPL: `shared`) |
| `images` | `base`, `sdk` |
| `recipe`, `patches` | Script; patches with `sha256` |
| `cpe` | CVE matching |
| `attestation` | `prebuilt` only: `{repo, signer_workflow}` |

`manifest_sha256` is the SHA-256 of canonical `sources.json` (`jq -S -c`). Each SBOM purl is `pkg:generic/<name>@<version>?checksum=sha256:<sha256>`.

## Build

Recipes build `/opt/vetload` in the pinned Amazon Linux 2023 builder, copied into the pinned `provided.al2023` runtime. Dynamic linking keeps LGPL libraries replaceable. Reproducibility: fixed paths, `SOURCE_DATE_EPOCH`, `-ffile-prefix-map`, deterministic `strip`, locked build-tool packages. Builds run natively on `ubuntu-24.04-arm` and `ubuntu-24.04`. PDFium is the one prebuilt input ([ADR-0001](docs/architecture/decisions/0001-pdfium-prebuilt-pinned-by-checksum.md)).

## ffmpeg configuration

FFmpeg 9.0.2, LGPL-2.1-or-later; every name checked against its 9.0.2 registries.

```sh
./configure --prefix=/opt/vetload --enable-shared --disable-static --enable-pic \
  --disable-everything --disable-autodetect --disable-network \
  --disable-doc --disable-debug --disable-avdevice --disable-ffplay \
  --enable-zlib --enable-libdav1d \
  --enable-protocol=file,pipe \
  --enable-demuxer=mov,matroska,mp3,aac,flac,wav,ogg,h264,hevc,ivf,obu \
  --enable-decoder=h264,hevc,vp8,vp9,libdav1d,mpeg4,prores,mjpeg,png,aac,mp3float,flac,vorbis,opus,alac,pcm_s16le,pcm_s16be,pcm_s24le,pcm_s24be,pcm_s32le,pcm_f32le,pcm_u8,pcm_alaw,pcm_mulaw \
  --enable-parser=h264,hevc,av1,vp8,vp9,mpeg4video,aac,mpegaudio,flac,opus,vorbis,mjpeg,png \
  --enable-bsf=h264_mp4toannexb,hevc_mp4toannexb,vp9_superframe_split \
  --enable-encoder=png,rawvideo,wrapped_avframe \
  --enable-muxer=image2pipe,rawvideo,null \
  --enable-filter=scale,aresample \
  --extra-ldexeflags='-Wl,-rpath,$ORIGIN/../lib'
```

| Enabled | Why |
| --- | --- |
| `ffmpeg`, `ffprobe`, `swscale`, `swresample` | Facts, frames, scaling, resampling |
| Protocols `pipe`, `file` | `pipe` for bytes and frames; `file` only for `/dev/fd/N`, a seekable in-memory descriptor. No network code |
| Demuxers `mov`, `matroska` | MP4, QuickTime, M4A; WebM |
| `mp3`, `aac`, `flac`, `wav`, `ogg` | Launch audio; `aac` is ADTS |
| `h264`, `hevc`, `ivf`, `obu` | Piped keyframe samples the engine range-read |
| Video decoders, `libdav1d` for AV1 | Posters; `mjpeg`, `png` also cover art |
| Audio decoders | Truncation checks; loudness later |
| Parsers | Parameters without decoding; raw-stream framing |
| Bitstream filters | MP4 samples to raw streams; `vp9` needs `vp9_superframe_split` |
| Encoders `png`, `rawvideo`, `wrapped_avframe` | Lossless posters; `wrapped_avframe` feeds `null` |
| Muxers `image2pipe`, `rawvideo`, `null` | Frames to stdout; decode-only passes |
| Filters `scale`, `aresample` | Bound frame size in the child; ffmpeg adds `format`, `null` and its rotation filters |
| `zlib`, `libdav1d` | PNG, compressed MP4 headers; AV1, shared with libheif |

Excluded: network protocols, playlist and concat demuxers, `image2`, devices, hardware acceleration, lossy encoders, GPL and non-free code. Additions need an issue and reason.

## Image layout

```text
/opt/vetload/bin/      vl-spawn ffmpeg ffprobe vips vipsheader qpdf vl-pdfium (P0) c2patool (P2)
/opt/vetload/lib/      shared libraries by SONAME: libvips.so.42, libheif.so.1, libpdfium.so, libqpdf, libav*
/opt/vetload/include/  headers and lib/pkgconfig/, sdk image only
/opt/vetload/share/vetload-native/  sources.json sbom.cdx.json build-info.json sizes.json ffmpeg-configure.txt licenses/
/opt/vetload/engine/   reserved for the engine image (C06)
/etc/ld.so.conf.d/vetload.conf
```

The image sets `VETLOAD_NATIVE_PREFIX=/opt/vetload`, `VETLOAD_NATIVE_RELEASE=<tag>`, `VIPS_BLOCK_UNTRUSTED=1` and `PATH`, keeping the base entrypoint. libvips has no modules and no PDF, SVG or ImageMagick loaders.

Images: `ghcr.io/vetload/native-base` and `ghcr.io/vetload/native-sdk` (plus headers). Tags `vMAJOR.MINOR.PATCH` never move: MAJOR removes a library, SONAME or path; MINOR adds or upgrades; PATCH rebuilds. Each tag is a multi-platform index; no `latest`.

## How C06 consumes it

Each release publishes `native-release.json`; `native.lock` embeds it unchanged (ADR-0049 E2):

```json
{
  "schema": 1,
  "release": "v0.1.0",
  "git_commit": "<sha1>",
  "manifest_sha256": "<sha256>",
  "images": { "native-base": {
    "repository": "ghcr.io/vetload/native-base",
    "index_digest": "sha256:…",
    "platforms": { "linux/arm64": "sha256:…", "linux/amd64": "sha256:…" } } },
  "assets": { "sbom.linux-arm64.cdx.json": "sha256:…" },
  "attestation": { "repo": "Vetload/vetload-native",
    "signer_workflow": "Vetload/vetload-native/.github/workflows/release.yml" }
}
```

C06 verifies attestation, builds `FROM ghcr.io/vetload/native-base@<index_digest>`, and reports `native_release` and `native_digest` in every result.

**`vl-spawn`** implements C06's ADR-0029 contract exactly: `vl-spawn --as BYTES --cpu SECONDS --nofile N --fsize BYTES [--seccomp nonet[:optional]] -- EXECUTABLE [ARG...]`. In order: `setsid`; `RLIMIT_CORE` 0 and the given limits (CPU hard one second above soft); `PR_SET_NO_NEW_PRIVS`; `PR_SET_PDEATHSIG` ignoring `EPERM`; the optional seccomp filter denying socket calls; `close_range(3, ~0U, 0)`; signal reset; `execve` without forking. Exits 125 launcher error, 126 not executable, 127 not found. C and libc only; hand-written BPF, no libseccomp.

**Tools** read stdin or `--input fd:N`, never a URL, and write one JSON document or image to stdout; stderr never carries file content. `vl-pdfium` exits 0 ok, 10 password required, 11 unsupported security handler, 12 limit exceeded, 65 malformed, 70 internal error.

## Interfaces provided

| Name | Kind | Identifier | Consumers | Phase |
| --- | --- | --- | --- | --- |
| Runtime image | image | `ghcr.io/vetload/native-base:vX.Y.Z@sha256:…` | C06, C11 to C15, C27 | Wave 1 arm64; P0 amd64 |
| Child launcher | release artefact | `/opt/vetload/bin/vl-spawn` (ADR-0029 contract) | C06 | Wave 1 |
| SDK image | image | `ghcr.io/vetload/native-sdk` | C06, C15 | P0 |
| Release descriptor | file format | `native-release.json`, `schemas/native-release.v1.json`, stable across `v0.x` | C06 (`native.lock`), C02 | Wave 1 |
| Layout, variables | image | `/opt/vetload/{bin,lib,share}`, `VETLOAD_NATIVE_PREFIX`, `VETLOAD_NATIVE_RELEASE` | C06, C11 to C13 | Wave 1 |
| Tools, libraries | release artefact | `ffmpeg`, `ffprobe`, `vips`, `vipsheader`, `qpdf`; `libvips.so.42`, `libheif`, `libpdfium`, `libqpdf`, `libav*` | C11 to C13 | Wave 1 |
| PDFium wrapper | release artefact | `vl-pdfium info`, `vl-pdfium render`; `schemas/vl-pdfium-info.v1.json` | C13, C15 | P0 |
| SBOM, provenance | release artefact | CycloneDX 1.6; GitHub build provenance | C15, C19, C27 | Wave 1; SBOM P0 |
| Size report | file format | `sizes.json` | C06 | P0 |

## Interfaces consumed

| Name | Kind | Identifier | Owner | Phase |
| --- | --- | --- | --- | --- |
| Child protocol and `vl-spawn` contract | C# interface | C06 ADR-0029 | C06 | Wave 1 |
| Result `engine` group | file format | `native_release`, `native_digest` | C01 | Wave 1 |
| Seed corpus | release artefact | `vetload-corpus-X.Y.Z.tar.gz`, for smoke tests | C04 | P0 |
| Sources, base images | release artefact, image | `sources.json` tarballs; AWS images in ADR-0002 | Upstream, AWS | Wave 1 |

## Data owned

GHCR packages `native-base` and `native-sdk`; releases `vX.Y.Z` with `native-release.json`, SBOMs, `sizes.json`, `SHA256SUMS` and, from P0, source mirrors. No AWS resources.

## Contract needs

1. Result `engine` group: `native_release` (`^v[0-9]+\.[0-9]+\.[0-9]+$`) and `native_digest` (`^sha256:[0-9a-f]{64}$`), filled by C06 from `native.lock`. Accepted (ADR-0049 A9).
2. `native.lock` embeds `native-release.json` unchanged, validated against `schemas/native-release.v1.json` here. Settled (ADR-0049 E2).
3. The `vl-pdfium` output schema and exit codes, for C13; canonical here. Open: Vetload/vetload-platform#32.

## Assumptions about other Wave 1 components

- **C01** adds the fields above.
- **C02** copies engine images to ECR by digest; no native builds in private CI.
- **C04**'s seed corpus covers HEIC, AVIF, PDF, MP4, WebM and launch audio.
- **C06** owns `native.lock`, spawns every child through `vl-spawn`, and measures cold start for the layer fallback.
- **C05, C07 to C09**: none.

## Failure modes

- A changed or vanished tarball fails the checksum; P0 adds mirrors.
- GPL or AGPL code linked into or enabled in anything C03 builds, an undeclared library, or an extra ffmpeg component fails the guard; base-OS programs are out of scope (ADR-0049 F3).
- Critical or high CVEs get a PATCH release within 72 hours or 7 days.
- If prebuilt PDFium stops, revisit ADR-0001.

## Security

Pinned, checksummed sources; base images by digest, bumped by Dependabot; actions pinned by commit; only the tag-triggered publish job gets `packages`, `id-token` and `attestations` write, with no secrets. Minimal decoders: no network, playlists, PDF JavaScript or plugins.

## Cost and scaling

Public CI, ARM included, and public GHCR packages are free: $0 at idle and at any volume. Stages cache independently, so a bump rebuilds only dependants; P3 sidecars reuse the manifest.

## Gate G1 deliverable

Tag `v0.1.0` publishing `ghcr.io/vetload/native-base` for `linux/arm64` with `vl-spawn`, libvips and codecs, libheif with libde265 and dav1d, lcms2, libexif, ffmpeg, ffprobe, PDFium and qpdf, plus `native-release.json` and provenance. Done when the tag's `crane digest` matches the descriptor, `gh attestation verify` passes, and the smoke job shows every tool starting under `vl-spawn`, its limit tests passing, and the expected libvips loaders. Consumer confirmations are follow-ups (ADR-0049 G1); the founder then makes the package public (F5).
