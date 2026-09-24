# Architecture: Vetload native stack (C03)

Builds the native libraries and tools Vetload's engine runs as child processes, from pinned, checksum-verified sources on public CI, published as images with digests, SBOM and provenance. Brief: `../vetload-platform/docs/components/C03-native-stack.md` (private).

## Modules

| Path | Purpose |
| --- | --- |
| `sources.json` | Source manifest |
| `recipes/<name>.sh` | Build script per manifest entry, with exact configure flags |
| `Dockerfile` | Stages `fetch` (download, verify), `build`, `base` and `sdk` |
| `wrappers/vl-pdfium/` | Small C program over PDFium's C API: structure dump and page render |
| `guard/` | Checksum verification, licence guard, component audit, size report, reproducibility comparison |
| `sbom/`, `schemas/` | CycloneDX generation; JSON Schemas for every file format below |
| `.github/workflows/` | `build` on pull requests, `release` on tags, scheduled positive controls |

## Source manifest: `sources.json`

Top level: `schema` (1), `bases` (builder and runtime images pinned by digest, [ADR-0002](docs/architecture/decisions/0002-base-operating-system-image.md)) and `sources`, an array of:

| Field | Meaning |
| --- | --- |
| `name`, `version` | SBOM component name (unique) and upstream version |
| `url` | HTTPS URL of the exact release tarball, never a generated git archive |
| `mirror` | Our copy as a release asset (P0), so rebuilds survive upstream removal |
| `sha256`, `sha256_origin` | Checksum the build enforces; `upstream-file`, `github-asset-digest` or `computed-after-signature` |
| `signature` | Optional `{url, key}`, verified whenever the pin changes |
| `license` | SPDX expression, read by the licence guard |
| `role`, `linkage` | `runtime`, `build-tool` or `prebuilt`; `shared`, `executable` or `build-only` (LGPL must be `shared`) |
| `images` | `base`, `sdk` |
| `recipe`, `patches` | Script path; patches with their own `sha256` |
| `cpe` | For CVE matching |
| `attestation` | `prebuilt` only: `{repo, signer_workflow}` for `gh attestation verify` |

The **manifest ID** is the SHA-256 of canonical `sources.json` (`jq -S -c`). Each SBOM purl is `pkg:generic/<name>@<version>?checksum=sha256:<sha256>`.

## Build

Recipes build into `/opt/vetload` in the pinned Amazon Linux 2023 builder; the tree is copied into the pinned `provided.al2023` runtime. Everything links dynamically, keeping LGPL libraries replaceable. Reproducibility: fixed build path, `SOURCE_DATE_EPOCH` from the commit, `-ffile-prefix-map`, `LC_ALL=C`, deterministic `strip`, and build tools from the builder's locked package repository. arm64 builds natively on `ubuntu-24.04-arm`, amd64 on `ubuntu-24.04`; no emulation. PDFium is the one prebuilt input ([ADR-0001](docs/architecture/decisions/0001-pdfium-prebuilt-pinned-by-checksum.md)).

## ffmpeg configuration

FFmpeg 9.0.2, LGPL-2.1-or-later. Every name was checked against the 9.0.2 component registries.

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
| `ffmpeg`, `ffprobe`, `swscale`, `swresample` | Facts; frame extraction; poster scaling; audio conversion |
| Protocols `pipe`, `file` | `pipe` streams bytes and frames; `file` only opens `/dev/fd/N`, a seekable in-memory descriptor for files indexed at the end. No network code |
| Demuxers `mov`, `matroska` | MP4, QuickTime, M4A; WebM, MKV next |
| `mp3`, `aac`, `flac`, `wav`, `ogg` | Launch audio kinds; `aac` is raw ADTS |
| `h264`, `hevc`, `ivf`, `obu` | The engine range-reads keyframe samples and pipes only those |
| Video decoders, `libdav1d` for AV1 | Poster frames for codecs in MP4, QuickTime, WebM; `mjpeg`, `png` also cover art |
| Audio decoders | Truncated-versus-malformed checks; loudness and waveforms later |
| Parsers | Stream parameters without decoding; frame boundaries in raw streams |
| Bitstream filters | MP4 samples to raw streams; `vp9` requires `vp9_superframe_split` |
| Encoders `png`, `rawvideo`, `wrapped_avframe` | Lossless posters to the resizer; `wrapped_avframe` feeds `null` |
| Muxers `image2pipe`, `rawvideo`, `null` | Frames to stdout; decode-only integrity passes |
| Filters `scale`, `aresample` | Bound frame size inside the child; ffmpeg auto-selects `format`, `null`, `trim`, `crop`, `rotate`, `transpose`, flips |
| `zlib`, `libdav1d` | PNG and compressed MP4 headers; AV1, shared with libheif |

Absent on purpose: network protocols, playlist and concat demuxers, `image2` (opens files by pattern), devices, hardware acceleration, lossy encoders, GPL and non-free code. C11 to C13 request additions by issue, with reasons.

## Image layout

```text
/opt/vetload/bin/      ffmpeg ffprobe vips vipsheader qpdf vl-pdfium (P0) c2patool (P2)
/opt/vetload/lib/      shared libraries by SONAME: libvips.so.42, libheif.so.1, libpdfium.so, libqpdf, libav*
/opt/vetload/include/  headers and lib/pkgconfig/, sdk image only
/opt/vetload/share/vetload-native/  sources.json sbom.cdx.json build-info.json sizes.json ffmpeg-configure.txt licenses/
/etc/ld.so.conf.d/vetload.conf
```

The image sets `VETLOAD_NATIVE_PREFIX=/opt/vetload`, `VETLOAD_NATIVE_RELEASE=<tag>`, `VIPS_BLOCK_UNTRUSTED=1` and `PATH`, and keeps the base entrypoint. libvips has no modules and no PDF, SVG or ImageMagick loaders, so PDFs reach only `vl-pdfium`.

Images: `ghcr.io/vetload/native-base` (runtime) and `ghcr.io/vetload/native-sdk` (plus headers). Tags `vMAJOR.MINOR.PATCH` are never moved: MAJOR removes a library, SONAME or path; MINOR adds or upgrades libraries; PATCH rebuilds or fixes. Each tag is a multi-platform index; no `latest` tag.

## How C06 consumes it

Each release publishes `native-release.json`; C06 copies it into `native.lock`:

```json
{
  "schema": 1,
  "release": "v0.1.0",
  "git_commit": "<40 hex>",
  "manifest_sha256": "<64 hex>",
  "images": { "native-base": {
    "repository": "ghcr.io/vetload/native-base",
    "index_digest": "sha256:<64 hex>",
    "platforms": { "linux/arm64": "sha256:<64 hex>", "linux/amd64": "sha256:<64 hex>" } } },
  "assets": { "sbom.linux-arm64.cdx.json": "sha256:<64 hex>" },
  "attestation": { "repo": "Vetload/vetload-native",
    "signer_workflow": "Vetload/vetload-native/.github/workflows/release.yml" }
}
```

C06 builds `FROM ghcr.io/vetload/native-base@<index_digest>` for `linux/arm64` after `gh attestation verify oci://…@<platform digest> --repo Vetload/vetload-native`, and reports release and digest in every result.

**Wrapper contract.** Input on stdin or a seekable descriptor (`--input fd:N`), never a URL; one JSON document or image on stdout; short diagnostics without file content on stderr. `vl-pdfium` exits 0 ok, 10 password required, 11 unsupported security handler, 12 limit exceeded, 65 malformed, 70 internal error.

## Interfaces provided

| Name | Kind | Identifier | Consumers | Phase |
| --- | --- | --- | --- | --- |
| Runtime image | image | `ghcr.io/vetload/native-base:vX.Y.Z@sha256:…` | C06, C11 to C15, C27 | Wave 1 arm64; P0 amd64 |
| SDK image | image | `ghcr.io/vetload/native-sdk` | C06, C15 for C shims | P0 |
| Release descriptor | file format | `native-release.json`, `schemas/native-release.v1.json` | C06, C02 | Wave 1 |
| Layout and variables | image | `/opt/vetload/{bin,lib,share}`, `VETLOAD_NATIVE_PREFIX`, `VETLOAD_NATIVE_RELEASE` | C06, C11 to C13 | Wave 1 |
| Tools | release artefact | `ffmpeg`, `ffprobe`, `vips`, `vipsheader`, `qpdf` | C11, C12, C13 | Wave 1 |
| Libraries | release artefact | `libvips.so.42`, `libheif`, `libpdfium`, `libqpdf`, `libav*` | C11, C13 | Wave 1 |
| PDFium wrapper | release artefact | `vl-pdfium info`, `vl-pdfium render`; `schemas/vl-pdfium-info.v1.json` | C13, C15 | P0 |
| SBOM, provenance | release artefact | CycloneDX 1.6 per platform; GitHub build provenance | C15, C19, C27 | Provenance Wave 1; SBOM P0 |
| Size report | file format | `sizes.json` | C06 | P0 |

## Interfaces consumed

| Name | Kind | Identifier | Owner | Phase |
| --- | --- | --- | --- | --- |
| Child-process conventions | C# interface | C06 child protocol: argv, pipes, descriptors, limits | C06 | Wave 1 |
| Result `engine` group | file format | Native release fields in the result schema | C01 | Wave 1 |
| Seed corpus | release artefact | Corpus release with checksums, for smoke tests | C04 | P0 |
| Sources and base images | release artefact, image | `sources.json` tarballs; `public.ecr.aws/lambda/provided:al2023`, `public.ecr.aws/amazonlinux/amazonlinux:2023` | Upstream, AWS | Wave 1 |

## Data owned

GHCR packages `ghcr.io/vetload/native-base` and `native-sdk`; releases `vX.Y.Z` of this repository with `native-release.json`, SBOMs, `sizes.json`, `SHA256SUMS` and, from P0, mirrored source tarballs; the Actions cache. No AWS resources.

## Contract needs

1. Result `engine` group: `native_release` (pattern `^v[0-9]+\.[0-9]+\.[0-9]+$`) and `native_digest` (`^sha256:[0-9a-f]{64}$`, platform image digest), filled by C06 from `native.lock`.
2. A home for the `native.lock` schema; canonical `native-release.v1` lives here, C01 decides whether `contracts/` mirrors it.
3. The `vl-pdfium` output schema and exit codes, for C13; canonical here.

## Assumptions about other Wave 1 components

- **C01** adds the fields above; nothing else crosses from C03.
- **C02** copies engine images to ECR per cell region by digest and builds no native code in private CI.
- **C04** publishes a seed corpus with checksums covering HEIC, AVIF, PDF, MP4, WebM and launch audio.
- **C06** owns `native.lock`, runs wrappers as plain command-line children with limits set before `exec`, and measures cold start for the layer fallback.
- **C05, C07 to C09**: no interface.

## Failure modes

- A changed or vanished upstream tarball fails the checksum; P0 adds mirrors.
- A GPL or undeclared linked library, or an extra ffmpeg component, fails the guard.
- A critical or high CVE triggers a PATCH release within 72 hours or 7 days.
- If prebuilt PDFium stops, ADR-0001 is revisited.

## Security

Pinned, checksummed sources; base images pinned by digest and bumped by Dependabot; actions pinned by commit; only the tag-triggered publish job gets `packages: write`, `id-token: write`, `attestations: write`, with no repository secrets. Minimal decoders: no network, playlists, PDF JavaScript or plugin loading.

## Cost and scaling

Public repository: Actions runners, ARM included, and public GHCR packages are free, so idle cost is $0 and inspection volume adds nothing; consumers pull by digest into their own ECR. Stages cache independently, so a library bump rebuilds only its dependants; P3 sidecars are further images from the same manifest.

## Gate G1 deliverable

Tag `v0.1.0` publishing `ghcr.io/vetload/native-base` for `linux/arm64` with libvips and codecs, libheif with libde265 and dav1d, lcms2, libexif, ffmpeg, ffprobe, PDFium and qpdf, plus `native-release.json` and provenance. Done when `crane digest ghcr.io/vetload/native-base:v0.1.0` matches the descriptor, `gh attestation verify` passes on it, and the tag's smoke job shows every tool starting and every expected libvips loader present.
