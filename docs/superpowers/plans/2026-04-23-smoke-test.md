# Docker Image Smoke Test Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add optional inline smoke test steps to the shared `docker-build.yml` reusable workflow, then wire up callers in `owncloud-docker/ocis` and `owncloud-docker/server`.

**Architecture:** Five new optional inputs are added to `docker-build.yml`. When `smoke-test-url` is non-empty, a new `Smoke test` step starts the already-built `127.0.0.1:5000/image:temp` image from the local registry, polls the URL until HTTP 200, and optionally asserts a version field matches `docker-tag`. All inputs are bound to `env:` vars in the `run:` block — never interpolated inline. The step is skipped when `smoke-test-url` is empty, so all existing callers are unaffected.

**Tech Stack:** GitHub Actions reusable workflows, Docker CLI, curl, jq, bash.

---

## File Map

| File | Repo | Action |
|------|------|--------|
| `.github/workflows/docker-build.yml` | `owncloud-docker/ubuntu` | Add 5 inputs + `Smoke test` step |
| `.github/workflows/main.yml` | `owncloud-docker/ocis` | Add smoke test inputs to `build` job |
| `.github/workflows/main.yml` | `owncloud-docker/server` | Add smoke test inputs + per-matrix `smoke-version-jq` field |

---

## Task 1: Add smoke test step to `owncloud-docker/ubuntu` docker-build.yml

**Files:**
- Modify: `owncloud-docker/ubuntu/.github/workflows/docker-build.yml` (remote repo — clone locally, commit, push, open PR)

**Context:** The workflow already builds the image into a local registry at `127.0.0.1:5000/image:temp` and has `network=host` on the Buildx driver, so the runner can reach that registry and any container port bound to localhost. The Trivy scan step already uses this image. The new `Smoke test` step goes immediately after Trivy, before `Set publish tags`.

The current file on `master` of `owncloud-docker/ubuntu` is:

```yaml
on:
  workflow_call:
    inputs:
      docker-repo-name:
        required: true
        type: string
      docker-tag:
        required: true
        type: string
      docker-context:
        required: true
        type: string
      docker-file:
        required: true
        type: string
      docker-hub-username:
        required: true
        type: string
      docker-build-args:
        required: false
        type: string
      push:
        required: true
        type: boolean
      trivy-ignore-files:
        required: false
        type: string
        default: ".trivyignore"
      docker-extra-tags:
        required: false
        type: string
        default: ""
    secrets:
      docker-hub-password:
        required: true
      docker-secrets:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest

    services:
      registry:
        image: registry:3
        ports:
          - 5000:5000

    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@4d04d5d9486b7bd6fa91e7baf45bbb4f8b9deedd # v4.0.0
        with:
          driver-opts: network=host

      - name: Build Docker Image
        id: docker-build
        uses: docker/build-push-action@bcafcacb16a39f128d818304e6c9c0c18556b85f # v7.1.0
        with:
          context: ${{ inputs.docker-context }}
          file: ${{ inputs.docker-file }}
          push: true
          platforms: linux/amd64,linux/arm64
          tags: 127.0.0.1:5000/image:temp
          secrets: ${{ secrets.docker-secrets }}
          build-args: ${{ inputs.docker-build-args }}

      - name: Trivy scan
        uses: aquasecurity/trivy-action@ed142fd0673e97e23eac54620cfb913e5ce36c25 # v0.36.0
        with:
          image-ref: 127.0.0.1:5000/image:temp
          hide-progress: true
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          skip-files: /usr/bin/gomplate,/usr/bin/wait-for
          exit-code: 1
          trivyignores: ${{ inputs.trivy-ignore-files }}

      - name: Set publish tags
        if: ${{ inputs.push }}
        id: tags
        env:
          DOCKER_REPO_NAME: ${{ inputs.docker-repo-name }}
          DOCKER_TAG: ${{ inputs.docker-tag }}
          DOCKER_EXTRA_TAGS: ${{ inputs.docker-extra-tags }}
        run: |
          TAGS="${DOCKER_REPO_NAME}:${DOCKER_TAG}"
          if [ -n "${DOCKER_EXTRA_TAGS}" ]; then
            while IFS= read -r tag; do
              [ -n "$tag" ] && TAGS="${TAGS}"$'\n'"${DOCKER_REPO_NAME}:${tag}"
            done <<< "${DOCKER_EXTRA_TAGS}"
          fi
          echo "list<<EOF" >> $GITHUB_OUTPUT
          echo "$TAGS" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      # Push the image on merge to master
      - name: Login to Docker Hub
        if: ${{ inputs.push }}
        uses: docker/login-action@4907a6ddec9925e35a0a9e82d7399ccc52663121 # v4.1.0
        with:
          username: ${{ inputs.docker-hub-username }}
          password: ${{ secrets.docker-hub-password }}

      - name: Publish image
        if: ${{ inputs.push }}
        uses: docker/build-push-action@bcafcacb16a39f128d818304e6c9c0c18556b85f # v7.1.0
        with:
          context: ${{ inputs.docker-context }}
          file: ${{ inputs.docker-file }}
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ steps.tags.outputs.list }}
          secrets: ${{ secrets.docker-secrets }}
          build-args: ${{ inputs.docker-build-args }}
```

- [ ] **Step 1: Check out the ubuntu repo locally**

If `/tmp/ubuntu-repo` already exists and is on `master`:
```bash
cd /tmp/ubuntu-repo
git checkout master
git pull origin master
```

Otherwise:
```bash
git clone https://github.com/owncloud-docker/ubuntu.git /tmp/ubuntu-repo
cd /tmp/ubuntu-repo
```

- [ ] **Step 2: Create a feature branch**

```bash
cd /tmp/ubuntu-repo
git checkout -b feat/smoke-test
```

Expected: `Switched to a new branch 'feat/smoke-test'`

- [ ] **Step 3: Write the updated `docker-build.yml`**

Write the complete file below to `/tmp/ubuntu-repo/.github/workflows/docker-build.yml`:

```yaml
on:
  workflow_call:
    inputs:
      docker-repo-name:
        required: true
        type: string
      docker-tag:
        required: true
        type: string
      docker-context:
        required: true
        type: string
      docker-file:
        required: true
        type: string
      docker-hub-username:
        required: true
        type: string
      docker-build-args:
        required: false
        type: string
      push:
        required: true
        type: boolean
      trivy-ignore-files:
        required: false
        type: string
        default: ".trivyignore"
      docker-extra-tags:
        required: false
        type: string
        default: ""
      smoke-test-port:
        required: false
        type: string
        default: ""
      smoke-test-url:
        required: false
        type: string
        default: ""
      smoke-test-env:
        required: false
        type: string
        default: ""
      smoke-test-entrypoint-cmd:
        required: false
        type: string
        default: ""
      smoke-test-version-jq:
        required: false
        type: string
        default: ""
    secrets:
      docker-hub-password:
        required: true
      docker-secrets:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest

    services:
      registry:
        image: registry:3
        ports:
          - 5000:5000

    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@4d04d5d9486b7bd6fa91e7baf45bbb4f8b9deedd # v4.0.0
        with:
          driver-opts: network=host

      - name: Build Docker Image
        id: docker-build
        uses: docker/build-push-action@bcafcacb16a39f128d818304e6c9c0c18556b85f # v7.1.0
        with:
          context: ${{ inputs.docker-context }}
          file: ${{ inputs.docker-file }}
          push: true
          platforms: linux/amd64,linux/arm64
          tags: 127.0.0.1:5000/image:temp
          secrets: ${{ secrets.docker-secrets }}
          build-args: ${{ inputs.docker-build-args }}

      - name: Trivy scan
        uses: aquasecurity/trivy-action@ed142fd0673e97e23eac54620cfb913e5ce36c25 # v0.36.0
        with:
          image-ref: 127.0.0.1:5000/image:temp
          hide-progress: true
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          skip-files: /usr/bin/gomplate,/usr/bin/wait-for
          exit-code: 1
          trivyignores: ${{ inputs.trivy-ignore-files }}

      - name: Smoke test
        if: ${{ inputs.smoke-test-url != '' }}
        env:
          SMOKE_PORT: ${{ inputs.smoke-test-port }}
          SMOKE_URL: ${{ inputs.smoke-test-url }}
          SMOKE_ENV: ${{ inputs.smoke-test-env }}
          SMOKE_ENTRYPOINT_CMD: ${{ inputs.smoke-test-entrypoint-cmd }}
          SMOKE_VERSION_JQ: ${{ inputs.smoke-test-version-jq }}
          DOCKER_TAG: ${{ inputs.docker-tag }}
        run: |
          ARGS="--rm -d"
          [ -n "${SMOKE_PORT}" ] && ARGS="${ARGS} -p ${SMOKE_PORT}:${SMOKE_PORT}"
          if [ -n "${SMOKE_ENV}" ]; then
            while IFS= read -r line; do
              [ -n "$line" ] && ARGS="${ARGS} --env ${line}"
            done <<< "${SMOKE_ENV}"
          fi
          if [ -n "${SMOKE_ENTRYPOINT_CMD}" ]; then
            CID=$(docker run ${ARGS} --entrypoint /bin/sh 127.0.0.1:5000/image:temp -c "${SMOKE_ENTRYPOINT_CMD}")
          else
            CID=$(docker run ${ARGS} 127.0.0.1:5000/image:temp)
          fi
          trap "docker stop ${CID} > /dev/null" EXIT
          echo "Container started: ${CID}"
          echo "Polling ${SMOKE_URL} ..."
          for i in $(seq 1 30); do
            STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "${SMOKE_URL}")
            if [ "${STATUS}" = "200" ]; then
              echo "Got HTTP 200 after $((i * 2))s"
              break
            fi
            if [ "${i}" = "30" ]; then
              echo "Timed out after 60s waiting for HTTP 200 (last status: ${STATUS})"
              docker logs "${CID}"
              exit 1
            fi
            sleep 2
          done
          if [ -n "${SMOKE_VERSION_JQ}" ]; then
            ACTUAL=$(curl -sk "${SMOKE_URL}" | jq -r "${SMOKE_VERSION_JQ}")
            if [ "${ACTUAL}" != "${DOCKER_TAG}" ]; then
              echo "Version mismatch: expected '${DOCKER_TAG}', got '${ACTUAL}'"
              exit 1
            fi
            echo "Version verified: ${ACTUAL}"
          fi

      - name: Set publish tags
        if: ${{ inputs.push }}
        id: tags
        env:
          DOCKER_REPO_NAME: ${{ inputs.docker-repo-name }}
          DOCKER_TAG: ${{ inputs.docker-tag }}
          DOCKER_EXTRA_TAGS: ${{ inputs.docker-extra-tags }}
        run: |
          TAGS="${DOCKER_REPO_NAME}:${DOCKER_TAG}"
          if [ -n "${DOCKER_EXTRA_TAGS}" ]; then
            while IFS= read -r tag; do
              [ -n "$tag" ] && TAGS="${TAGS}"$'\n'"${DOCKER_REPO_NAME}:${tag}"
            done <<< "${DOCKER_EXTRA_TAGS}"
          fi
          echo "list<<EOF" >> $GITHUB_OUTPUT
          echo "$TAGS" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      # Push the image on merge to master
      - name: Login to Docker Hub
        if: ${{ inputs.push }}
        uses: docker/login-action@4907a6ddec9925e35a0a9e82d7399ccc52663121 # v4.1.0
        with:
          username: ${{ inputs.docker-hub-username }}
          password: ${{ secrets.docker-hub-password }}

      - name: Publish image
        if: ${{ inputs.push }}
        uses: docker/build-push-action@bcafcacb16a39f128d818304e6c9c0c18556b85f # v7.1.0
        with:
          context: ${{ inputs.docker-context }}
          file: ${{ inputs.docker-file }}
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ steps.tags.outputs.list }}
          secrets: ${{ secrets.docker-secrets }}
          build-args: ${{ inputs.docker-build-args }}
```

- [ ] **Step 4: Verify the diff**

```bash
cd /tmp/ubuntu-repo
git diff .github/workflows/docker-build.yml
```

Expected diff shows:
- 5 new inputs added after `docker-extra-tags`: `smoke-test-port`, `smoke-test-url`, `smoke-test-env`, `smoke-test-entrypoint-cmd`, `smoke-test-version-jq`
- New `Smoke test` step added between `Trivy scan` and `Set publish tags`
- No other changes

- [ ] **Step 5: Commit (signed)**

```bash
cd /tmp/ubuntu-repo
git add .github/workflows/docker-build.yml
git commit -S -m "feat: add optional smoke test step to docker-build workflow

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

Verify signed:
```bash
git log --show-signature -1 2>&1 | grep -E 'Good signature|gpg:'
```

Expected: `gpg: Good signature from ...`

- [ ] **Step 6: Push the branch**

```bash
cd /tmp/ubuntu-repo
git remote set-url origin "https://x-access-token:$(gh auth token)@github.com/owncloud-docker/ubuntu.git"
git push origin feat/smoke-test
```

Expected: branch pushed, no errors.

- [ ] **Step 7: Open a PR on `owncloud-docker/ubuntu`**

```bash
gh pr create \
  --repo owncloud-docker/ubuntu \
  --head feat/smoke-test \
  --base master \
  --title "feat: add optional smoke test step to docker-build workflow" \
  --body "$(cat <<'EOF'
## Summary

- Adds 5 optional inputs to \`docker-build.yml\`: \`smoke-test-port\`, \`smoke-test-url\`, \`smoke-test-env\`, \`smoke-test-entrypoint-cmd\`, \`smoke-test-version-jq\`
- When \`smoke-test-url\` is empty (default), the \`Smoke test\` step is skipped — no change for existing callers
- When set, the step starts \`127.0.0.1:5000/image:temp\` from the local registry, polls the URL every 2s for up to 60s, and fails with container logs on timeout
- \`smoke-test-entrypoint-cmd\` overrides the entrypoint to \`/bin/sh -c \"<cmd>\"\` for images that need an init step (e.g. oCIS: \`ocis init || true; exec ocis server\`)
- \`smoke-test-version-jq\` optionally asserts a jq-extracted field from the response JSON equals \`docker-tag\`
- All inputs are bound to \`env:\` vars in the \`run:\` block — no inline expression interpolation

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Expected: PR URL printed.

- [ ] **Step 8: Verify the PR diff on GitHub**

```bash
gh pr diff --repo owncloud-docker/ubuntu feat/smoke-test
```

Confirm no unintended changes beyond the 5 inputs and the `Smoke test` step.

---

## Task 2: Update `owncloud-docker/ocis` main.yml with smoke test inputs

**Files:**
- Modify: `.github/workflows/main.yml` in local repo at `/home/deepdiver/Development/claude-work/ocis-docker`

**Context:** Must be done after Task 1's ubuntu PR is merged — the reusable workflow must accept the new inputs before callers can pass them. Work on the existing open branch `feat/extra-tags` which already has the matrix `extra-tags` changes and the `CODEOWNERS` file.

The `build` job's `with:` block currently ends with:
```yaml
      docker-extra-tags: ${{ matrix.release.extra-tags }}
      push: false
```

The matrix currently has:
```yaml
        release:
          - version: "7.3.2"
            dir: "v7"
            extra-tags: |
              7.3
              7
          - version: "8.0.1"
            dir: "v8"
            extra-tags: |
              8.0
              8
```

- [ ] **Step 1: Check out the feature branch**

```bash
cd /home/deepdiver/Development/claude-work/ocis-docker
git checkout feat/extra-tags
git pull origin feat/extra-tags
```

- [ ] **Step 2: Update `.github/workflows/main.yml`**

The full file after the change:

```yaml
name: Docker CI

on:
  push:
    branches: [master]
    tags: ["*"]
  pull_request:
  schedule:
    - cron: 0 0 * * 0

jobs:
  lint:
    uses: owncloud-docker/ubuntu/.github/workflows/lint-editorconfig.yml@master

  build:
    needs: lint
    uses: owncloud-docker/ubuntu/.github/workflows/docker-build.yml@master
    with:
      docker-repo-name: owncloud/${{ github.event.repository.name }}
      docker-tag: ${{ matrix.release.version }}
      docker-context: ${{ matrix.release.dir }}
      docker-file: ${{ matrix.release.dir }}/Dockerfile.multiarch
      docker-hub-username: ${{ vars.DOCKERHUB_USERNAME }}
      docker-build-args: |
        VERSION=${{ matrix.release.version }}
        REVISION=${{ github.sha }}
      trivy-ignore-files: .trivyignore,${{ matrix.release.dir }}/.trivyignore
      docker-extra-tags: ${{ matrix.release.extra-tags }}
      smoke-test-port: "9200"
      smoke-test-url: "https://localhost:9200/health"
      smoke-test-entrypoint-cmd: "ocis init || true; exec ocis server"
      smoke-test-env: |
        OCIS_INSECURE=true
        OCIS_URL=https://localhost:9200
      push: false
    secrets:
      docker-hub-password: ${{ secrets.DOCKERHUB_TOKEN }}

    strategy:
      matrix:
        release:
          - version: "7.3.2"
            dir: "v7"
            extra-tags: |
              7.3
              7
          - version: "8.0.1"
            dir: "v8"
            extra-tags: |
              8.0
              8

  # update-docker-hub-description is disabled until this repo is authorized to publish
  # update-docker-hub-description:
  #   needs: build
  #   if: github.ref == 'refs/heads/master'
  #   uses: owncloud-docker/ubuntu/.github/workflows/docker-hub-desc.yml@master
  #   with:
  #     docker-repo-name: owncloud/${{ github.event.repository.name }}
  #     docker-repo-description: ownCloud Infinite Scale
  #     docker-hub-username: ${{ vars.DOCKERHUB_USERNAME }}
  #   secrets:
  #     docker-hub-password: ${{ secrets.DOCKERHUB_TOKEN }}
```

- [ ] **Step 3: Commit (signed)**

```bash
cd /home/deepdiver/Development/claude-work/ocis-docker
git add .github/workflows/main.yml
git commit -S -m "feat: add smoke test inputs to CI workflow

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

- [ ] **Step 4: Push**

```bash
git push origin feat/extra-tags
```

Expected: push succeeds, new commit visible on the open PR#3.

---

## Task 3: Update `owncloud-docker/server` main.yml with smoke test inputs

**Files:**
- Modify: `owncloud-docker/server/.github/workflows/main.yml` (remote repo — clone locally, commit, push, open PR)

**Context:** Must be done after Task 1's ubuntu PR is merged. The server matrix has 4 entries; 3 stable versions use `smoke-test-version-jq: ".versionstring"` to assert the version, and `11.0.0-prealpha` omits it (daily tarball version string is unpredictable). Each matrix entry gains a `smoke-version-jq` field; the `build` job passes `smoke-test-version-jq: ${{ matrix.release.smoke-version-jq }}`.

The server image exposes port 8080 over HTTP. `/status.php` returns JSON like:
```json
{"installed":true,"maintenance":false,"needsDbUpgrade":false,"version":"10.16.2.3","versionstring":"10.16.2","edition":"Community","productname":"ownCloud","hostname":""}
```

The `smoke-test-env` needs `OWNCLOUD_DOMAIN=localhost:8080` and `OWNCLOUD_DB_TYPE=sqlite3` to allow the server to start without an external database.

- [ ] **Step 1: Clone the server repo (or update if already cloned)**

```bash
git clone https://github.com/owncloud-docker/server.git /tmp/server-repo
```

Or if already cloned:
```bash
cd /tmp/server-repo && git checkout master && git pull origin master
```

- [ ] **Step 2: Create a feature branch**

```bash
cd /tmp/server-repo
git checkout -b feat/smoke-test
```

- [ ] **Step 3: Write the updated `main.yml`**

Write this complete file to `/tmp/server-repo/.github/workflows/main.yml`:

```yaml
name: Docker CI

on:
  push:
    branches: [master]
    tags: ["*"]
  pull_request:
  schedule:
    - cron: 0 0 * * 0

jobs:
  lint:
    uses: owncloud-docker/ubuntu/.github/workflows/lint-editorconfig.yml@master

  build:
    needs: lint
    uses: owncloud-docker/ubuntu/.github/workflows/docker-build.yml@master
    with:
      docker-repo-name: owncloud/${{ github.event.repository.name }}
      docker-tag: ${{ matrix.release.version }}
      docker-context: ${{ matrix.release.base }}
      docker-file: ${{ matrix.release.base }}/Dockerfile.multiarch
      docker-hub-username: ${{ vars.DOCKERHUB_USERNAME }}
      docker-build-args: |
        TARBALL_URL=${{ matrix.release.tarball }}
      push: ${{ github.ref == 'refs/heads/master' }}
      trivy-ignore-files: ${{ matrix.release.trivy-ignore }}
      docker-extra-tags: ${{ matrix.release.extra-tags }}
      smoke-test-port: "8080"
      smoke-test-url: "http://localhost:8080/status.php"
      smoke-test-env: |
        OWNCLOUD_DOMAIN=localhost:8080
        OWNCLOUD_DB_TYPE=sqlite3
      smoke-test-version-jq: ${{ matrix.release.smoke-version-jq }}
    secrets:
      docker-hub-password: ${{ secrets.DOCKERHUB_TOKEN }}

    strategy:
      matrix:
        release:
          - version: 10.16.2
            tarball: https://download.owncloud.com/server/stable/owncloud-complete-20260422.tar.bz2
            base: v22.04
            trivy-ignore: v22.04/10.16.2/.trivyignore
            extra-tags: |
              10.16
              10
              latest
            smoke-version-jq: ".versionstring"
          - version: 10.16.1
            tarball: https://download.owncloud.com/server/stable/owncloud-complete-20260218.tar.bz2
            base: v22.04
            trivy-ignore: v22.04/10.16.1/.trivyignore
            smoke-version-jq: ".versionstring"
          - version: 10.15.3
            tarball: https://download.owncloud.com/server/stable/owncloud-complete-20250703.tar.bz2
            base: v22.04
            trivy-ignore: v22.04/10.15.3/.trivyignore
            extra-tags: |
              10.15
            smoke-version-jq: ".versionstring"
          - version: 11.0.0-prealpha
            tarball: https://download.owncloud.com/server/daily/owncloud-daily-master.tar.bz2
            base: v24.04
            trivy-ignore: v24.04/11.0.0-prealpha/.trivyignore
            smoke-version-jq: ""

  update-docker-hub-description:
    needs: build
    if: github.ref == 'refs/heads/master'
    uses: owncloud-docker/ubuntu/.github/workflows/docker-hub-desc.yml@master
    with:
      docker-repo-name: owncloud/${{ github.event.repository.name }}
      docker-repo-description: ownCloud - Secure Collaboration Platform
      docker-hub-username: ${{ vars.DOCKERHUB_USERNAME }}
    secrets:
      docker-hub-password: ${{ secrets.DOCKERHUB_TOKEN }}
```

- [ ] **Step 4: Verify the diff**

```bash
cd /tmp/server-repo
git diff .github/workflows/main.yml
```

Expected diff shows:
- 3 new fixed inputs added to `with:` block: `smoke-test-port`, `smoke-test-url`, `smoke-test-env`
- New `smoke-test-version-jq: ${{ matrix.release.smoke-version-jq }}` line in `with:`
- New `smoke-version-jq` field added to each of the 4 matrix entries (3 with `".versionstring"`, 1 with `""`)
- No other changes

- [ ] **Step 5: Commit (signed)**

```bash
cd /tmp/server-repo
git add .github/workflows/main.yml
git commit -S -m "feat: add smoke test inputs to CI workflow

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

Verify signed:
```bash
git log --show-signature -1 2>&1 | grep -E 'Good signature|gpg:'
```

Expected: `gpg: Good signature from ...`

- [ ] **Step 6: Push the branch**

```bash
cd /tmp/server-repo
git remote set-url origin "https://x-access-token:$(gh auth token)@github.com/owncloud-docker/server.git"
git push origin feat/smoke-test
```

- [ ] **Step 7: Open a PR on `owncloud-docker/server`**

```bash
gh pr create \
  --repo owncloud-docker/server \
  --head feat/smoke-test \
  --base master \
  --title "feat: add smoke test inputs to CI workflow" \
  --body "$(cat <<'EOF'
## Summary

- Passes smoke test inputs to the reusable \`docker-build.yml\` workflow (requires owncloud-docker/ubuntu smoke-test PR merged first)
- Starts the built image on port 8080 with SQLite3 and polls \`/status.php\` for HTTP 200
- Asserts \`.versionstring\` in the response JSON matches the matrix version for all stable releases
- \`11.0.0-prealpha\` skips version assertion (\`smoke-version-jq: ""\`) since the daily tarball version is unpredictable

## Behaviour per matrix entry

| Version | HTTP 200 check | Version assertion |
|---------|---------------|-------------------|
| 10.16.2 | yes | \`.versionstring\` == \`10.16.2\` |
| 10.16.1 | yes | \`.versionstring\` == \`10.16.1\` |
| 10.15.3 | yes | \`.versionstring\` == \`10.15.3\` |
| 11.0.0-prealpha | yes | skipped |

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Expected: PR URL printed.
