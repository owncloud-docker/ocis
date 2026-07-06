# ownCloud Infinite Scale Docker Image

[![Docker Pulls](https://img.shields.io/docker/pulls/owncloud/ocis.svg)](https://hub.docker.com/r/owncloud/ocis)
[![License: Apache-2.0](https://img.shields.io/github/license/owncloud-docker/ocis)](https://github.com/owncloud-docker/ocis/blob/master/LICENSE)
[![ownCloud OSPO](https://img.shields.io/badge/OSPO-ownCloud-blue)](https://kiteworks.com/opensource)

Docker image for [ownCloud Infinite Scale (oCIS)](https://github.com/owncloud/ocis) — a modern file-sync and share platform.

> **Build & maintenance:** see [how these images are built, scanned, updated and published](https://github.com/owncloud-docker/.github/blob/master/docs/IMAGE-LIFECYCLE.md).

## Quick Start

```bash
docker run --rm \
  -p 9200:9200 \
  -e OCIS_INSECURE=true \
  owncloud/ocis:8.0.1
```

## Supported Tags

| Tag | oCIS Version |
|-----|-------------|
| `8.0.5`, `8.0`, `8` | 8.0.5 (latest stable) |
| `8.1.0-rc.2` | 8.1.0-rc.2 (release candidate — exact tag only, no floating tags) |
| `<version>-YYYYMMDD` | Immutable per-build tag (e.g. `8.0.5-20260623`) |

## Rolling Image

A daily build tracking the **unreleased `master` branch** of oCIS is published to
[`owncloud/ocis-rolling`](https://hub.docker.com/r/owncloud/ocis-rolling). It is built
every night at 02:00 UTC from the latest `master` commit.

> **Unstable:** the rolling image contains unreleased code and is intended for testing
> against upcoming oCIS changes — not for production.

| Tag | Meaning |
|-----|---------|
| `latest` | The most recent daily build |
| `YYYYMMDD` | Immutable build for a specific day (e.g. `20260602`) |
| `sha-<short>` | Build of a specific oCIS `master` commit (e.g. `sha-a1b2c3d`) |

```bash
docker pull owncloud/ocis-rolling:latest
```

## Volumes

| Path | Purpose |
|------|---------|
| `/var/lib/ocis` | Data directory |
| `/etc/ocis` | Configuration directory |

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| `9200` | TCP | HTTPS gateway |

## Build Arguments

| ARG | Default | Purpose |
|-----|---------|---------|
| `VERSION` | version-specific | oCIS git tag to clone and build (without `v` prefix, e.g. `8.0.1`) |
| `GIT_REF` | `v${VERSION}` | Git ref (branch or tag) to clone and build; overrides the default tag form to build a branch such as `master` |
| `GIT_SHA` | `""` | Optional exact commit to check out after cloning `GIT_REF`. Pins branch builds to a resolved commit and busts the clone-layer cache when it changes, keeping rolling builds fresh. Empty for release builds. |
| `REVISION` | `""` | Git SHA embedded in OCI labels |
| `TARGETARCH` | set by buildx | Target architecture (`amd64`, `arm64`) |

## Building

The image is built entirely from source via a three-stage Dockerfile:

**`node-builder`** — clones the oCIS git repository at `v${VERSION}`, builds the IDP React frontend (`pnpm build`) and downloads the web frontend assets (`make pull-assets`). Both are required at compile time because `services/idp` and `services/web` use `//go:embed`. The IDP `pnpm` build runs only when `services/idp/package.json` is present; newer oCIS `master` ("no-npm") commits the identifier assets directly and has no `package.json`, so the step is skipped automatically.

**`go-builder`** — compiles the oCIS binary with CGO and libvips enabled using the upstream Makefile target `release-linux-docker-${TARGETARCH}`. Outputs to `dist/binaries/ocis-linux-${TARGETARCH}`.

**Runtime** — minimal Alpine image with the binary copied from `go-builder`. The stage runs `apk upgrade` to refresh all installed OS packages to the latest available Alpine patch releases at build time, so security fixes are picked up immediately rather than waiting for a base-image tag bump.

To build locally:

```bash
docker buildx build \
  --build-arg VERSION=8.0.1 \
  --build-arg REVISION=$(git rev-parse HEAD) \
  --platform linux/amd64 \
  -f v8/Dockerfile.multiarch v8/
```

## CI

The GitHub Actions workflow (`.github/workflows/main.yml`) builds and validates the image on every push, pull request, and weekly schedule.

**Steps per release matrix entry:**

1. **Build** — multi-arch image (`linux/amd64`, `linux/arm64`) pushed to an ephemeral local registry using BuildKit with GHA layer cache.
2. **Trivy scan** — scans for HIGH/CRITICAL CVEs; unfixable upstream CVEs are listed in `v8/.trivyignore`.
3. **Smoke test** — starts the container, polls `https://localhost:9200/status.php` every 2s for up to 62s, and verifies the `.productversion` field in the JSON response matches the built tag. Uses `OCIS_INSECURE=true` to allow self-signed TLS on the test runner.
4. **Publish** — pushes to Docker Hub with floating major/minor tags (on `master` only).

## Community & Support

- [ownCloud Website](https://owncloud.com)
- [Community Discussions](https://github.com/orgs/owncloud/discussions)
- [Matrix Chat](https://app.element.io/#/room/#owncloud:matrix.org)
- [Documentation](https://doc.owncloud.com)
- [Enterprise Support](https://owncloud.com/contact-us/)
- [OSPO Home](https://kiteworks.com/opensource)

See [SUPPORT.md](SUPPORT.md) for the full list of support channels.

## Contributing

We welcome contributions! Please read the [Contributing Guidelines](CONTRIBUTING.md)
and our [Code of Conduct](CODE_OF_CONDUCT.md) before getting started.

- **Rebase Early, Rebase Often!** We use a rebase workflow — rebase on the target
  branch before submitting a PR.
- **Signed commits**: All commits **must** be PGP/GPG signed and carry a DCO
  `Signed-off-by` line (`git commit -S -s`).
- **Conventional Commits**: PR titles must follow the
  [Conventional Commits](https://www.conventionalcommits.org/) format — enforced
  by CI.
- **GitHub Actions Policy**: Workflows may only use actions owned by `owncloud`,
  created by GitHub (`actions/*`), or verified in the GitHub Marketplace, pinned
  to a full commit SHA.

Note: bug reports and feature requests for oCIS itself belong upstream in
[`owncloud/ocis`](https://github.com/owncloud/ocis) — this repository tracks only
the Docker packaging.

## Security

**Do not open a public GitHub issue for security vulnerabilities.**

Report vulnerabilities at **<https://security.owncloud.com>** — see [SECURITY.md](SECURITY.md).

Bug bounty: [YesWeHack ownCloud Program](https://yeswehack.com/programs/owncloud-bug-bounty-program)

## About the ownCloud OSPO

The [Kiteworks Open Source Program Office](https://kiteworks.com/opensource), operating under
the [ownCloud](https://owncloud.com) brand, launched on May 5, 2026, to steward the open source
ecosystem around ownCloud's products. The OSPO ensures transparent governance, license compliance,
community health, and sustainable collaboration between the open source community and
[Kiteworks](https://www.kiteworks.com), which acquired ownCloud in 2023.

- **OSPO Home**: <https://kiteworks.com/opensource>
- **GitHub**: <https://github.com/owncloud>
- **ownCloud**: <https://owncloud.com>

For questions about the OSPO or licensing, contact ospo@kiteworks.com.

This repository is licensed under the **Apache License 2.0**, which is the license
the OSPO is adopting across the ecosystem. No relicensing is required.

## License

Apache-2.0 — see [LICENSE](LICENSE).
