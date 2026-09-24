# Implementation plan: Vetload native stack (C03)

Companion to [ARCHITECTURE.md](ARCHITECTURE.md). Each milestone is one or more small pull requests. A milestone is done only when its checks have been seen failing once and then passing, with the failing run linked in the pull request.

## Milestones

| Milestone | Phase | Outcome |
| --- | --- | --- |
| M0 Planning | Wave 1 | This pull request: architecture, plan, ADR-0001 and ADR-0002, and a spike that probes the GitHub-hosted runners |
| M1 First base image | Wave 1, gate G1 | `v0.1.0`: `ghcr.io/vetload/native-base` for `linux/arm64`, with provenance |
| M2 Production images | P0 | Both architectures, `native-sdk`, `vl-pdfium`, SBOM, full licence guard, size budget, reproducibility check, source mirrors |
| M3 Codecs and CVE watch | P1 | libultrahdr, libavif and an AVIF encoder; scheduled vulnerability scan against the SBOM |
| M4 Provenance tools | P2 | `c2patool`; ffmpeg additions for sprites and hover clips |
| M5 Sidecars and compliance | P3 | LibRaw, FreeType, HarfBuzz, DCMTK; LibreOffice and ClamAV sidecar base images; LGPL compliance bundle; assurance pack with C15 |

## M1: first base image (gate G1)

1. `schemas/sources.v1.json` and `sources.json` with the entries in the seed manifest below; a CI job validates the manifest against the schema.
2. `guard/fetch-verify.sh`: downloads each `url` over HTTPS, checks `sha256` before extraction, and refuses anything else. For entries with `sha256_origin: computed-after-signature`, the pin is recorded only after the upstream signature verifies against a key fingerprint stored in the manifest.
3. Recipes for zlib, libffi, pcre2, glib, expat, libjpeg-turbo, libpng, libwebp, libtiff, lcms2, libexif, highway, libde265, dav1d, libheif, libvips, qpdf and ffmpeg, plus the PDFium unpack step from ADR-0001.
   - glib: without libmount, SELinux, introspection or tests.
   - libheif: libde265 and dav1d decoders only; no encoders, no plugin loading, no examples.
   - libvips: modules off; loaders for JPEG, PNG, WebP, TIFF, GIF (built-in), HEIF, with lcms2, libexif and highway; PDF, SVG, ImageMagick, OpenSlide, Poppler, libimagequant and every other optional loader off.
   - qpdf: built-in crypto only, so there is no OpenSSL or GnuTLS dependency; confirm the option names against 12.4.1's CMake files.
   - ffmpeg: exactly the configure line in ARCHITECTURE.md.
4. `Dockerfile` with pinned builder and runtime digests, the `/opt/vetload` layout, `ld.so.conf.d` entry and environment variables.
5. Guards needed for G1:
   - **Component audit**: `ffprobe -hide_banner -demuxers`, `-decoders`, `-protocols`, `-bsfs`, `-filters` and `-muxers` must equal the lists committed in `guard/ffmpeg-expected/`; `vips -l foreign` must contain the expected loaders and none of the excluded ones.
   - **Licence guard, first version**: manifest SPDX allowlist; `ffmpeg -L` must report LGPL version 2.1 or later; every `DT_NEEDED` entry of every ELF file under `/opt/vetload` must resolve to a manifest library or an allowlisted base-OS library such as glibc, libstdc++ and libgcc_s.
   - **ABI floor**: no ELF under `/opt/vetload` requires a newer `GLIBC_` symbol version than the runtime image provides; this matters most for prebuilt PDFium.
6. `release.yml`: triggered by `v*` tags on `main`, builds on `ubuntu-24.04-arm`, pushes by digest, refuses to overwrite an existing tag, writes `native-release.json` and `SHA256SUMS`, and runs `actions/attest-build-provenance` on the image digest.
7. Smoke job on the published digest: every tool starts; `vipsheader` reads a JPEG, PNG, WebP and TIFF that `vips` writes in the job; `qpdf --empty` produces a PDF that a PDFium smoke program from a test-only stage opens; `ffprobe` reads a WAV that the runner writes and pipes in. HEIC, AVIF and MP4 coverage waits for the C04 corpus, because the image deliberately has no encoders for them.
8. Tag `v0.1.0`; post the digest and the verification commands on the kickoff issue.

## M2: production images (P0)

1. `linux/amd64` build on `ubuntu-24.04`, and a multi-platform index made with `docker buildx imagetools create`.
2. `native-sdk` image: the same tree plus `include/` and `lib/pkgconfig/`.
3. `vl-pdfium` in C: `info` (version, page count and sizes, form type, JavaScript action count, attachment names, encryption and permissions, title and author, outline presence, tagged, language, signature count) and `render --page N --max-side PX` producing PPM. Input from stdin or `--input fd:N`, page and pixel limits, and the exit codes in ARCHITECTURE.md. The JSON Schema `schemas/vl-pdfium-info.v1.json` is validated in CI against its output on every PDF in the corpus. A block-request input mode is added only if C13 asks for it (open question 3).
4. CycloneDX 1.6 SBOM generated from the manifest, with ffmpeg's configure line and enabled component lists as properties, merged with a Syft scan of the base OS layer; attached as a release asset and as an attestation.
5. Full licence guard: the M1 checks, plus LGPL entries must have `linkage: shared` and ship a `.so`, plus a report listing every base-OS package whose licence is GPL or AGPL, which must match a reviewed allowlist (open question 1).
6. Size report `sizes.json` per library and per image, with a budget of **100 MB for `/opt/vetload` in the base image**. That keeps the default path far below the 250 MB unzipped limit of a Lambda layer, so the zip-and-layer fallback in the platform's packaging decision stays open. The figure is an estimate to be confirmed by the first measured build.
7. Reproducibility: every release builds the arm64 tree twice on separate runners and compares SHA-256 of every file; differences fail the release unless explained in `guard/repro-allowlist.txt` with a reason.
8. Source mirrors: each release uploads its verified tarballs as assets, and `mirror` points at them.
9. Corpus smoke: run `ffprobe`, `vipsheader` and `vl-pdfium info` over the C04 seed corpus; no crash, and outputs are parsed.
10. Dependabot for Docker digests and GitHub Actions; `gitleaks` in CI.
11. If C06 measures a p95 cold start over one second, publish the default-path tree as a layer zip asset built from the same stage.

## M3 to M5

- **M3 (P1)**: libultrahdr 2.0.2 (probed), libavif 1.4.2 (probed) with an AVIF encoder chosen by licence and size in an ADR; scheduled Grype scan of each release's SBOM with OpenVEX statements for compiled-out code, opening issues with the 72-hour and 7-day targets; ffmpeg additions requested by C12 for P1, such as `avi` and `thumbnail`; a from-source PDFium build as a check on ADR-0001.
- **M4 (P2)**: `c2patool` built from its pinned source release, and ffmpeg additions for sprites and hover clips (`tile`, `fps`, libvpx, libwebp animation), each with a reason and a new audit list.
- **M5 (P3)**: LibRaw, FreeType, HarfBuzz and DCMTK; LibreOffice and ClamAV sidecar base images, with ClamAV only in its own image; LGPL source offer and relinking notes; the supply-chain assurance pack with C15.

## Seed manifest (probed 2026-09-24)

Checksums marked *asset* are the SHA-256 digests GitHub reports for release assets; *upstream* came from the project's published checksum file; *pending* means no checksum is published, so M1 computes it only after the upstream signature verifies.

| Name | Version | SHA-256 (origin) | Licence |
| --- | --- | --- | --- |
| zlib | 1.3.2 (`.tar.xz`) | `d7a0654783a4da529d1bb793b7ad9c3318020af77667bcae35f95d0e42a792f3` (asset) | Zlib |
| libffi | 3.8.0 | `7da3e2d9a171eb0a038f592ecad3ff2bb2550f3496d87b3b29ad0cf4430c0db4` (asset) | MIT |
| pcre2 | 10.48 (`.tar.bz2`) | `b6c68fdf6f3ac31388b50aa89ff0fc49c00c987c16e7b5146491d12003f2c8ed` (asset) | BSD-3-Clause WITH PCRE2-exception |
| glib | 2.88.3 | `ab24d24e698dfa1e408b7bcdb508f4aafc906185a8b8ce72fdf79bbbdc9b383b` (upstream) | LGPL-2.1-or-later |
| expat | 2.8.5 (`.tar.xz`) | `1e727b8933ec51a77a9a9d9afcf8e688bce45d907c13e36ab7393fe36e703182` (asset) | MIT |
| libjpeg-turbo | 3.2.0 | `6f30092cef9fb839779646608f4ee14ae3cbac989c47fa05e841b0841f09878e` (asset) | IJG AND BSD-3-Clause AND Zlib |
| libpng | 1.6.58 | pending | libpng-2.0 |
| libwebp | 1.6.0 | pending (signature `.asc`) | BSD-3-Clause |
| libtiff | 4.7.2 (`.tar.xz`, 2,409,652 bytes) | pending (signature `.sig`) | libtiff |
| lcms2 | 2.19.1 | `bfc54f7bab59fbc921012014a8032e4cba4abd46db47d46b76416a8c0b2815c8` (asset) | MIT |
| libexif | 0.6.26 (`.tar.xz`) | `4a055ed6575e61ca46c3172be3c753cc16c9becd0f99ec71d58dd0e471476c0c` (asset) | LGPL-2.1-or-later |
| highway | 1.4.0 | `36f672ab48ddb3c8555e9e89e16fe400cd7d16c6eb455a1a3d0c146a63ababdc` (asset) | Apache-2.0 OR BSD-3-Clause |
| libde265 | 1.1.3 | `554228bd17788c99a7e63b37ab5634722190e6e2bf60c1dcb01cef328e133905` (asset) | LGPL-3.0-or-later |
| dav1d | 1.5.4 | `686616b7c69eb88d44459391ab25cac13b6647a3b288835c5784e71c1514a5c5` (upstream) | BSD-2-Clause |
| libheif | 1.23.5 | `fd9036064c4432f0550d15072ddf34956a248279ee9aeaff0fba3fa0f77d8f1a` (asset) | LGPL-3.0-or-later |
| libvips | 8.18.6 | `3c41e1d5458081bfa4a5bc54e116c46259c75c6760a18027764555632b9dda3e` (asset) | LGPL-2.1-or-later |
| qpdf | 12.4.1 | `f045aa277be2356ff53a89a8622945958291177d2483afc20ede7c8a8cd3873c` (asset) | Apache-2.0 |
| ffmpeg | 9.0.2 (`.tar.xz`, 12,040,788 bytes) | pending (signature `.asc`) | LGPL-2.1-or-later |
| pdfium (prebuilt) | `chromium/8066` | arm64 `0e6f90dccbc6b81fd5d7106abaf164c4222178f024c204d00d526b60fd2ad535`, amd64 `0b43f405477cf2cfc4dbff06905093c3309756c6bca1fb9da99234a2ca97fed2` (asset) | BSD-3-Clause plus bundled third-party licences |

Base images, pinned by index digest: `public.ecr.aws/lambda/provided:al2023@sha256:c6a2e7cb103884b785ebce3ce826c7cfd68e9843d8d5e6773a53e902ced83a19` and `public.ecr.aws/amazonlinux/amazonlinux:2023@sha256:06da5a3362eda00c5114227ac81abe56dd9b943395553b7c64a310457f4d9e2b`. Build tools (meson, ninja, cmake, nasm, a C and C++ compiler) come from the builder's locked package repository; any that are missing or too old become `build-tool` manifest entries. Versions will move before M1 lands; the manifest pull request re-probes them.

## Test plan

Every check below has a negative case that must fail, run on every pull request unless marked scheduled.

| Check | Must fail when | Positive control |
| --- | --- | --- |
| Manifest schema | A required field is missing | Fixture manifest with `sha256` removed |
| Checksum verification | Any byte of a tarball or pin differs | One hex digit flipped in a temporary copy of the manifest; the fetch must stop before extraction |
| Signature verification (on pin change) | Signature does not match the key | Tarball verified against a wrong fingerprint |
| Licence guard, manifest | A licence is outside the allowlist | Fixture entry `GPL-2.0-or-later` |
| Licence guard, linkage | An ELF needs a library that is neither in the manifest nor allowlisted | Tiny fixture ELF linked against a dummy `libfixture.so` |
| Licence guard, ffmpeg | ffmpeg reports GPL | Scheduled weekly build of ffmpeg with `--enable-gpl`, and once in the M2 pull request; the guard must reject it |
| Licence guard, nothing found | The guard scanned zero ELF files | The guard prints the count and fails on zero |
| ffmpeg component audit | Any extra or missing demuxer, decoder, protocol, filter, muxer or bitstream filter | Expected list with one entry removed |
| No network in ffmpeg | `ffprobe http://127.0.0.1/` or an HLS playlist fixture is opened | The same harness opens a valid local file through `pipe:0`, proving it can observe success |
| libvips loaders | `pdfload`, `svgload` or `magickload` present, or `heifload` missing | Expected list with `heifload` removed |
| PDFium without V8 | `libpdfium.so` exports V8 symbols or needs a V8 library | The same check run once against the upstream V8 variant must fail |
| ABI floor | An ELF requires a newer `GLIBC_` version than the runtime has | Fixture ELF with a raised requirement |
| Size budget (P0) | `/opt/vetload` exceeds 100 MB | Budget set to 1 MB in a test run |
| Reproducibility (P0) | Two builds differ without an explanation | A test run injects a timestamp into one file |
| Tag immutability | A release would overwrite an existing tag | Dry run against `v0.1.0` after it exists |
| Provenance | The attestation does not verify | Verification against a wrong `--repo` must fail |
| Corpus smoke (P0) | A tool crashes or output does not parse | The job fails if it processed zero files |

## Risks

| Risk | Mitigation |
| --- | --- |
| C12 or C13 needs ffmpeg components not listed | Additions by issue; the audit list changes in the same pull request |
| Implementation notes suggest ffprobe reading signed URLs over TLS | Conflicts with the brief and the platform security baseline; this plan compiles no network code and relies on the engine piping bytes (open question 4) |
| Prebuilt PDFium supply chain | Checksum plus SLSA provenance verification; from-source check in P1; ADR-0001 |
| Build time within the 6-hour job limit | Per-stage caching; first build is measured in M1. Disk is ample: the spike measured 108 GB free on `ubuntu-24.04-arm`, although GitHub documents 14 GB |
| GHCR packages start private | Founder action below |
| Upstream CVE cadence, especially in image codecs | Scheduled scan from P1; patch releases are small because stages are cached |
| Reproducibility gaps in some build systems | Explained differences are allowed but listed with a reason |

## Open questions

1. The brief fixes "nothing GPL or AGPL in the engine images", while this repository's README says "linked or enabled". The runtime base image contains a shell and other operating-system programs, some of which are GPL, as separate programs. This plan treats the rule as "never linked or enabled in anything we build", and lists base-OS GPL packages in the SBOM against a reviewed allowlist. The founder should confirm (ADR-0002).
2. Does C06's child protocol accept plain command-line children (argv, stdin, stdout, exit code), or do wrappers need a framed mode?
3. Does C13 need `vl-pdfium` to request byte ranges from the supervisor, or is whole-file input under a size gate enough?
4. C12: the video implementation notes propose ffprobe over HTTPS with GnuTLS. This plan follows the brief (pipe and file only). Confirm with C12 in Wave 2.
5. Does anyone need `native-sdk` before P0?
6. Does C04 need an image with encoders for its generators? `native-base` has no lossy encoders by design.

## Founder actions

- After the first push, make the `native-base` and `native-sdk` packages on GHCR **public** and link them to this repository. GHCR creates packages private by default, and the organisation must allow public packages.
- Confirm open question 1.
- Enable branch protection on `main` and, if available, immutable releases for this repository. Branch protection is free for public repositories.
