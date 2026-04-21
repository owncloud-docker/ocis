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
| `7.3.2` | 7.3.2 |

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
| `VERSION` | version-specific | oCIS release to embed |
| `REVISION` | `""` | Git SHA embedded in OCI labels |

## License

Apache-2.0 — see [LICENSE](LICENSE).
