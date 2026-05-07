# ownCloud Infinite Scale Docker Image

[![Docker Pulls](https://img.shields.io/docker/pulls/owncloud/ocis.svg)](https://hub.docker.com/r/owncloud/ocis)

Docker image for [ownCloud Infinite Scale (oCIS)](https://github.com/owncloud/ocis) — a modern file-sync and share platform.

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
| `8.0.1` | 8.0.1 |

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
| `REVISION` | `""` | Git SHA embedded in OCI labels |
| `TARGETARCH` | set by buildx | Target architecture (`amd64`, `arm64`) |

## Building

The image is built entirely from source via a three-stage Dockerfile:

**`node-builder`** — clones the oCIS git repository at `v${VERSION}`, builds the IDP React frontend (`pnpm build`) and downloads the web frontend assets (`make pull-assets`). Both are required at compile time because `services/idp` and `services/web` use `//go:embed`.

**`go-builder`** — compiles the oCIS binary with CGO and libvips enabled using the upstream Makefile target `release-linux-docker-${TARGETARCH}`. Outputs to `dist/binaries/ocis-linux-${TARGETARCH}`.

**Runtime** — minimal Alpine image with the binary copied from `go-builder`.

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

## License

Apache-2.0 — see [LICENSE](LICENSE).
