# 0002. Base operating system image: Lambda `provided.al2023` for runtime, Amazon Linux 2023 for building

- **Status:** proposed
- **Date:** 2026-09-24
- **Component:** C03

## Context

The platform's packaging decision makes engine functions ARM64 container images built from this repository's base image and pinned by digest. The same image must run on Lambda, on Lambda Managed Instances, on Fargate Spot and in private deployments. If p95 cold start exceeds one second, the default path falls back to a zip package with a Lambda layer. The brief leaves open whether the base is Lambda's `provided.al2023` image or a minimal Amazon Linux 2023 image. Facts probed on 2026-09-24 with `crane`:

| Image | Index digest | arm64 digest | arm64 compressed size | Notes |
| --- | --- | --- | --- | --- |
| `public.ecr.aws/lambda/provided:al2023` | `sha256:c6a2e7cb103884b785ebce3ce826c7cfd68e9843d8d5e6773a53e902ced83a19` | `sha256:a4a1e72fb03a80c19d6eb34013f8efdcca8e29e9eb7fd2c69eea5a9d50cd48a3` | 39 MB, 4 layers | Entrypoint `/lambda-entrypoint.sh`; `PATH` includes `/opt/bin`; `LD_LIBRARY_PATH` includes `/opt/lib`; created 2026-09-24 |
| `public.ecr.aws/amazonlinux/amazonlinux:2023-minimal` | `sha256:68b1e82cd69ade271c092bc1959fa6fc820945d0c8dbb7a88cf6c1824a184688` | `sha256:60e925a21bd208f3298ba2b5faaa3a41a5a3d1e0a2f10f1fde5b2f5b72334903` | 34 MB, 1 layer | No entrypoint |
| `public.ecr.aws/amazonlinux/amazonlinux:2023` | `sha256:06da5a3362eda00c5114227ac81abe56dd9b943395553b7c64a310457f4d9e2b` | `sha256:9ffc413368b2ca84207418e1853705be207d7aa74ea11f918895969d4041e098` | 51 MB, 1 layer | Full package manager; builder candidate |

AWS states that `provided.al2023` is based on the Amazon Linux 2023 minimal image. Amazon Linux 2023 is supported until June 2029 (Amazon Linux 2023 FAQ). Both images publish `linux/arm64` and `linux/amd64`.

## Decision

- **Runtime stage: `public.ecr.aws/lambda/provided:al2023`, pinned by index digest.** The published `native-base` and `native-sdk` images are built on it and keep its entrypoint. The engine sets its own command. Other runtimes override the entrypoint.
- **Builder stage: `public.ecr.aws/amazonlinux/amazonlinux:2023`, pinned by index digest.** Toolchain packages come from its locked repository release. The build records every installed package version in `build-info.json` and asserts that nothing under `/opt/vetload` needs a newer `GLIBC_` version than the runtime provides.
- Both digests live in `sources.json` under `bases`. **Dependabot** opens pull requests when AWS publishes new digests. Each bump is an ordinary reviewed change that produces a PATCH release.
- **GPL scope, as decided by the founder in [ADR-0049](https://github.com/Vetload/vetload-platform/blob/main/docs/architecture/decisions/0049-wave-1-alignment.md) F3.** The rule covers only what C03 builds, links or enables. No library or tool that C03 builds for the engine images may link or enable GPL or AGPL code. The licence guard enforces this on C03's own build outputs under `/opt/vetload`. Operating-system programs that ship with the pinned base image, such as the shell that runs the entrypoint script and coreutils, are outside the rule and outside the guard. They are still listed in the SBOM for transparency. ClamAV stays in its own sidecar image.

## Options considered

| Option | For | Against |
| --- | --- | --- |
| `provided.al2023` runtime (chosen) | Exactly the OS of Lambda's managed `provided.al2023` runtime, so the same `/opt/vetload` tree also works as a layer for the zip fallback; includes, per AWS documentation, the runtime interface emulator for local invocation tests; AWS-maintained weekly | About 5 MB more than minimal; Lambda-specific entrypoint to override elsewhere |
| `amazonlinux:2023-minimal` runtime | Smallest; neutral across runtimes | The engine must add Lambda's runtime interface pieces for local tests; slightly more drift from the managed runtime used by the zip fallback |
| A non-Amazon distribution such as Debian slim or distroless | Familiar; distroless has no shell | Different glibc from Lambda's managed runtime breaks the layer fallback; mixes OS families across products |

## Consequences

- One tree serves both the image and a possible layer, so ADR-0013 in the platform repository (the packaging decision) can switch the default path without a rebuild.
- The runtime image's glibc version is the compatibility floor. Amazon Linux 2023 is expected to ship glibc 2.34; M1 confirms this. Prebuilt inputs such as PDFium are checked against the floor.
- Base-image updates are frequent. Each costs one free CI build and one platform lock bump.
- Cost: $0. Public images, public CI.

## Revisit when

- AWS publishes a successor to Amazon Linux 2023 for Lambda, or support approaches its end in June 2029.
- A superseding program ADR changes the GPL scope set by ADR-0049 F3.
- Cold-start measurements show the runtime base image itself matters.
