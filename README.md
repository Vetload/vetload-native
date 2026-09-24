# Vetload native stack

Pinned, checksum-verified, reproducible builds of the native media libraries Vetload uses to inspect files: ffmpeg and ffprobe, libvips and its codecs, libheif, PDFium, qpdf and more. Every library is built from a source tarball whose SHA-256 is recorded here, so anyone can audit exactly what parses their files.

**Status:** being set up. The manifest, build recipe and first images arrive in the first build wave.

## Outputs

- Base container images for `linux/arm64` and `linux/amd64`, published with digests.
- A CycloneDX SBOM and build provenance for every image.
- A licence guard that fails the build if any GPL or AGPL component is linked or enabled.

## Licence

Build scripts are licensed under Apache-2.0 (see `LICENSE`). Each library keeps its own licence, recorded in the manifest.
