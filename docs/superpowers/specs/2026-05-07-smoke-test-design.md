# oCIS Docker Image: Smoke Test

**Date:** 2026-05-07
**Scope:** Add a smoke test to the `build` job in `.github/workflows/main.yml` that verifies the built oCIS image is functional before publishing.

## Problem

The CI pipeline builds and Trivy-scans the image but never runs it. A broken binary, a missing embedded asset, or a wrong embedded version would not be caught until after the image is published.

## Design

Wire the two smoke-test mechanisms already provided by `owncloud-docker/ubuntu/.github/workflows/docker-build.yml` into the existing `build` job. No new jobs or files are required.

### Inputs added to the `build` job

| Input | Value | Purpose |
|---|---|---|
| `smoke-test-cmd` | `/usr/bin/ocis version` | One-shot binary check: confirms binary is present and executable |
| `smoke-test-port` | `9200` | Exposes the oCIS HTTPS port for polling |
| `smoke-test-url` | `https://localhost:9200/status.php` | Endpoint polled for HTTP 200 |
| `smoke-test-entrypoint-cmd` | `ocis init \|\| true; exec ocis server` | Initialises config then starts the server as PID 1 |
| `smoke-test-env` | `OCIS_INSECURE=true` | Disables inter-service TLS verification for single-node test deployments |
| `smoke-test-version-jq` | `.versionstring` | Extracts the semver string from the `/status.php` JSON response and asserts it equals `docker-tag` |

### Execution order (within the reusable workflow)

1. **Build** — multi-arch image pushed to local ephemeral registry
2. **Trivy scan**
3. **Smoke test (cmd)** — `docker run --rm image /bin/sh -c "/usr/bin/ocis version"`; exits 0 or fails the job
4. **Smoke test (server)** — starts container with `ocis init || true; exec ocis server`, polls `https://localhost:9200/status.php` every 2s for up to 62s, verifies `.versionstring == docker-tag`

Steps 3 and 4 are independent guards — both run if both inputs are set.

### Constraints

- **Startup time:** The workflow polls for 62s (31 × 2s). oCIS starts ~20 microservices; 62s is generally sufficient on standard GitHub runners.
- **TLS:** oCIS generates a self-signed cert on first run. `OCIS_INSECURE=true` disables inter-service TLS verification; `curl -sk` in the workflow skips cert validation for the external poll.
- **Permissions:** The container runs as uid 1000 (`ocis-user`). `/var/lib/ocis` and `/etc/ocis` are already `chown`ed to that user in the Dockerfile. `ocis init` writes to both directories without permission issues.
- **Ephemeral state:** The smoke test container is started with `--rm -d` and torn down after the poll. No volumes are mounted.
- **Architecture:** The smoke test runs on `ubuntu-latest` (amd64). The arm64 binary is built and Trivy-scanned but not smoke-tested — this is acceptable.

## Files Changed

- `.github/workflows/main.yml` — add six inputs to the `with:` block of the `build` job's `uses:` call
