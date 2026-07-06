# agents.md — ocis

## Repository Overview

This repository builds the official **ownCloud Infinite Scale (oCIS)** Docker
images (`owncloud/ocis` on Docker Hub, plus the daily `owncloud/ocis-rolling`).
It does not contain the oCIS source code — it builds oCIS **from source** via a
multi-stage Dockerfile and ships a minimal Alpine runtime image. Images are
multi-architecture and built via GitHub Actions.

- **Classification:** Docker image build (from source)
- **Activity Status:** Active
- **License:** Apache-2.0
- **Language:** Dockerfile, Shell

## Architecture & Key Paths

- `v8/` — oCIS 8.x build context
  - `v8/Dockerfile.multiarch` — three-stage build:
    1. **node-builder** — clones oCIS at `v${VERSION}`, builds the IDP React
       frontend (`pnpm build`) and pulls the web assets (`make pull-assets`);
       both are needed at compile time due to `//go:embed`. The `pnpm` step is
       skipped automatically on newer `master` ("no-npm") commits.
    2. **go-builder** — compiles the oCIS binary with CGO + libvips via the
       upstream `release-linux-docker-${TARGETARCH}` Makefile target.
    3. **runtime** — minimal Alpine image; runs `apk upgrade` at build time so
       OS security fixes are picked up immediately.
  - `v8/.trivyignore` — accepted-CVE exclusions for the Trivy scan
- `ubuntu-wf/` — helper directory
- `docs/` — design/spec notes
- `.github/workflows/main.yml` — **active** CI: builds the release matrix
- `.github/workflows/rolling.yml` — nightly build tracking oCIS `master`
- `.github/workflows/lint-pr-title.yml` — Conventional-Commit PR-title enforcement
- `.github/dependabot.yml` — weekly GitHub Actions dependency updates
- `.github/CODEOWNERS` — review ownership
- `.renovaterc.json` — Renovate preset for Docker digest updates
- `.editorconfig` — formatting rules (2-space indent, LF, trailing newline)
- `.trivyignore` — repo-root accepted-CVE exclusions
- `LICENSE` — Apache-2.0

There is **no `CHANGELOG.md`** in this repository.

## Build & CI

The image is built entirely from source. CI (`main.yml`) calls the reusable
`docker-build-native.yml` workflow hosted in
[`owncloud-docker/ubuntu`](https://github.com/owncloud-docker/ubuntu):

- Release matrix builds tagged oCIS versions (e.g. `8.0.5` → `8.0`, `8`;
  `8.1.0-rc.2` exact tag only), plus immutable `<version>-YYYYMMDD` tags.
- `rolling.yml` resolves oCIS `master` HEAD nightly and publishes
  `owncloud/ocis-rolling` (`latest`, `YYYYMMDD`, `sha-<short>`). `VERSION` is
  left empty on purpose so the build derives the version from the git HEAD.
- Smoke test: poll `https://localhost:9200/status.php` (with `OCIS_INSECURE=true`)
  and assert `.productversion` matches the built tag.
- Trivy vulnerability scan (`.trivyignore` + `v8/.trivyignore`).
- On non-PR events: push to Docker Hub and sync the README as the description.

To build locally:

```bash
docker buildx build \
  --build-arg VERSION=8.0.5 \
  --build-arg REVISION=$(git rev-parse HEAD) \
  --platform linux/amd64 \
  -f v8/Dockerfile.multiarch v8/
```

The image exposes port `9200` and uses volumes `/var/lib/ocis`, `/etc/ocis`.

## Development Conventions

- **No CHANGELOG** — do not create one.
- Conventional-Commit PR titles, enforced by `lint-pr-title.yml`.
- `.editorconfig` governs formatting.
- GitHub Actions are pinned to full commit SHAs.
- Bug reports for oCIS itself go upstream to
  [`owncloud/ocis`](https://github.com/owncloud/ocis); this repo tracks only the
  Docker packaging.

## OSPO Policy Constraints

### GitHub Actions
- **Only** use actions owned by `owncloud`, created by GitHub (`actions/*`),
  verified on the GitHub Marketplace, or verified by the ownCloud Maintainers.
- Pin all actions to their full commit SHA (not tags): `uses: actions/checkout@<SHA> # vX.Y.Z`.
- Never introduce actions from unverified third parties.

### Dependency Management
- Dependabot is configured for GitHub Actions updates; Renovate handles Docker
  base-image digest updates.
- Review and merge dependency PRs as part of regular maintenance.

### Git Workflow
- **Rebase policy**: Always rebase; never create merge commits.
- **Signed commits**: All commits **must** be PGP/GPG signed (`git commit -S`).
- **DCO sign-off**: Every commit needs a `Signed-off-by` line (`git commit -s`).
- **Conventional Commits & Squash Merge**: PR titles must follow
  [Conventional Commits](https://www.conventionalcommits.org/); the PR title
  becomes the squash-merge commit message and is enforced by CI.

## Context for AI Agents

- This is a Docker-image build repo that compiles oCIS from source — not the oCIS
  application codebase.
- The active build systems are `main.yml` (releases) and `rolling.yml` (nightly
  master); the rolling build intentionally leaves `VERSION` empty.
- The README is published verbatim as the Docker Hub image description — keep it
  accurate and self-contained.
- License is **Apache-2.0**, which is the OSPO's ecosystem-wide migration target;
  no relicensing is required.
