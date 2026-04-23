# Docker Image Smoke Test — Design

## Overview

Add inline smoke test steps to the shared `docker-build.yml` reusable workflow in `owncloud-docker/ubuntu`. After the Trivy scan, a new optional `Smoke test` step starts the just-built image from the local registry (`127.0.0.1:5000/image:temp`), polls an HTTP/HTTPS endpoint until it responds with 200, and optionally asserts that a version field in the JSON response matches the matrix version. The step is skipped entirely when `smoke-test-url` is not set, so existing callers are unaffected.

---

## New Inputs (all optional, default `""`)

| Input | Purpose | Example |
|-------|---------|---------|
| `smoke-test-port` | Container port to map to the same host port | `"9200"` |
| `smoke-test-url` | URL to poll; step skipped when empty | `"https://localhost:9200/health"` |
| `smoke-test-env` | Newline-separated `KEY=VALUE` env vars for the container | `"OCIS_INSECURE=true\nOCIS_URL=https://localhost:9200"` |
| `smoke-test-entrypoint-cmd` | When set, overrides entrypoint to `/bin/sh -c "<cmd>"` | `"ocis init \|\| true; exec ocis server"` |
| `smoke-test-version-jq` | jq expression to extract version from response JSON; when set, asserts value equals `docker-tag` | `".versionstring"` |

---

## Smoke Test Step Behaviour

Placed after `Trivy scan`, before `Set publish tags`. Runs unconditionally (not gated on `inputs.push`).

1. **Build `docker run` args** from env vars (all inputs bound to `env:`, never interpolated inline)
   - `--rm -d -p ${SMOKE_PORT}:${SMOKE_PORT}` 
   - For each line in `$SMOKE_ENV`: add `--env KEY=VALUE`
   - If `$SMOKE_ENTRYPOINT_CMD` is non-empty: `--entrypoint /bin/sh … -c "$SMOKE_ENTRYPOINT_CMD"`
2. **Start container**, capture container ID
3. **Trap** `docker stop $CID` to guarantee cleanup on success or failure
4. **Poll** `curl -sk -o /dev/null -w "%{http_code}" $SMOKE_URL` every 2 s for up to 60 s; on timeout print `docker logs $CID` and exit 1
5. **If `$SMOKE_VERSION_JQ` is set**: fetch the URL, parse with `jq -r "$SMOKE_VERSION_JQ"`, assert equals `$DOCKER_TAG`; mismatch prints both values and exits 1

---

## Caller Configuration

### `owncloud-docker/ocis` — `main.yml`

```yaml
smoke-test-port: "9200"
smoke-test-url: "https://localhost:9200/health"
smoke-test-entrypoint-cmd: "ocis init || true; exec ocis server"
smoke-test-env: |
  OCIS_INSECURE=true
  OCIS_URL=https://localhost:9200
```

No `smoke-test-version-jq` — oCIS's `/health` endpoint does not expose a plain version field in a stable JSON structure.

### `owncloud-docker/server` — `main.yml`

```yaml
smoke-test-port: "8080"
smoke-test-url: "http://localhost:8080/status.php"
smoke-test-version-jq: ".versionstring"
smoke-test-env: |
  OWNCLOUD_DOMAIN=localhost:8080
  OWNCLOUD_DB_TYPE=sqlite3
```

No entrypoint override — server has its own `/usr/bin/entrypoint` wrapper.

**Note on `11.0.0-prealpha`:** The version matrix entry for prealpha uses a daily tarball; its `versionstring` will not match a clean semver. That entry must omit `smoke-test-version-jq` (set to `""`) or override it per matrix entry. The plan will handle this with a per-matrix `smoke-version-jq` field.

---

## Files Changed

| File | Repo | Action |
|------|------|--------|
| `.github/workflows/docker-build.yml` | `owncloud-docker/ubuntu` | Add 5 inputs + `Smoke test` step |
| `.github/workflows/main.yml` | `owncloud-docker/ocis` | Add smoke test inputs to `build` job |
| `.github/workflows/main.yml` | `owncloud-docker/server` | Add smoke test inputs to `build` job (per-matrix for version-jq) |

---

## Out of Scope

- Authenticated endpoint tests
- Multi-container startup (database, Redis)
- Any test beyond HTTP 200 + optional version string match
