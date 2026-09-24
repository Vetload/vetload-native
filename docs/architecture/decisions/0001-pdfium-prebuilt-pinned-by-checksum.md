# 0001. PDFium: pin a well-known prebuilt release by checksum and provenance

- **Status:** proposed
- **Date:** 2026-09-24
- **Component:** C03

## Context

The engine renders and inspects PDFs through PDFium, run as a child process behind the `vl-pdfium` wrapper. The brief leaves open whether to build PDFium from source or to pin a well-known prebuilt release by checksum. Facts probed on 2026-09-24:

- **PDFium has no source tarballs.** It is developed in Chromium's git infrastructure and checked out with `gclient`, which also fetches a Chromium toolchain (clang, GN, sysroots). A source build needs that toolchain. Our "pinned source tarball" would be a snapshot we made ourselves, not an upstream artefact.
- **Runners:** standard GitHub-hosted arm64 runners for public repositories have 4 cores, 16 GB of RAM and 14 GB of SSD, and a job may run for at most 6 hours (GitHub documentation, read 2026-09-24). A PDFium checkout with its toolchain takes a significant share of that disk. We have not measured it.
- **`bblanchon/pdfium-binaries`** (MIT-licensed build scripts, public GitHub Actions builds) publishes weekly builds per Chromium branch. Release `chromium/8066` (PDFium 156.0.8066.0, published 2026-09-21) has these assets:
  - `pdfium-linux-arm64.tgz`: 3,664,840 bytes, SHA-256 `0e6f90dccbc6b81fd5d7106abaf164c4222178f024c204d00d526b60fd2ad535`
  - `pdfium-linux-x64.tgz`: 3,743,765 bytes, SHA-256 `0b43f405477cf2cfc4dbff06905093c3309756c6bca1fb9da99234a2ca97fed2`
- The GitHub attestations API returns **two attestations for the arm64 asset**: SLSA provenance v1 from `.github/workflows/build-all.yml` on `refs/heads/master` of `bblanchon/pdfium-binaries`, and an in-toto release attestation.
- The project's `steps/05-configure.sh` sets `pdf_enable_v8 = false` and `pdf_enable_xfa = false` for the non-V8 variant, with `is_debug = false`, `pdf_is_standalone = true`, `is_component_build = false` and `pdf_use_partition_alloc = false`. The platform requires PDFium to be built without JavaScript or XFA execution.

## Decision

- Use the **non-V8 `pdfium-linux-arm64.tgz` and `pdfium-linux-x64.tgz`** from `bblanchon/pdfium-binaries`. Record them in `sources.json` with `role: prebuilt`, the `chromium/NNNN` tag as the version, each asset's SHA-256, and `attestation: {repo: "bblanchon/pdfium-binaries", signer_workflow: "bblanchon/pdfium-binaries/.github/workflows/build-all.yml"}`.
- The `fetch` stage accepts the archive only if **both** the SHA-256 matches and `gh attestation verify <file> --repo bblanchon/pdfium-binaries --signer-workflow …` passes.
- The build asserts that `libpdfium.so` has no V8 symbols and needs no V8 library. It also asserts that its maximum required `GLIBC_` version is supported by the runtime image, and that the licence files in the archive are copied to `share/vetload-native/licenses/pdfium/`.
- Our own code, `vl-pdfium`, is built from source here against the headers in the same archive.
- In **P1**, a scheduled job builds the same Chromium branch from source with the same GN arguments on our runners. It compares exported symbols and the output of `vl-pdfium` over the corpus, and records the build time and disk use. It is a check, not the shipping path.
- A PDFium upgrade means moving to a newer `chromium/NNNN` release: a manifest pull request with the new checksums and a passing attestation check.

## Options considered

| Option | For | Against |
| --- | --- | --- |
| Build from source with `gclient` in our CI | Everything auditable from Google's git; our own GN arguments | No upstream tarball to pin; large checkout on a 14 GB runner; hours of work on the gate G1 critical path; we would maintain a Chromium toolchain setup |
| Prebuilt from `bblanchon/pdfium-binaries`, pinned by checksum and provenance (chosen) | Small (under 4 MB compressed); public build scripts; verifiable SLSA provenance; the non-V8 variant matches the no-JavaScript requirement; both architectures | Trust in one maintainer's pipeline; we inherit their build flags and timing; Chromium's bundled libc++ is linked inside |
| Distribution packages | Simple | Amazon Linux 2023 does not ship PDFium; other distributions' packages would break the "pinned and verified" rule |

## Consequences

- G1 does not depend on getting a Chromium toolchain working on ARM runners.
- Supply-chain trust extends to a third-party GitHub workflow. Provenance verification limits this to "built by that workflow from that repository", and the P1 source build tests it independently.
- PDFium CVE fixes arrive when upstream publishes a new weekly release. If the maintainer is slow, the 72-hour target for critical CVEs is at risk. The fallback is the P1 source build.
- Cost: $0. A few megabytes per release, fetched on public CI.

## Revisit when

- `bblanchon/pdfium-binaries` stops publishing attestations, changes the non-V8 build flags, or goes more than 14 days without a release while a critical PDFium CVE is open.
- The P1 source build fits within one standard runner job, which would make building ourselves cheap enough to switch.
- We need GN arguments the prebuilt does not offer, such as disabling specific decoders like JBIG2 or JPX.
