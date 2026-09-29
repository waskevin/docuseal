# DocuSeal for TerraMaster TOS 7

TOS 7 App Center package for [DocuSeal](https://www.docuseal.com) — an open-source platform to fill, sign and manage document templates.

一个把 [DocuSeal](https://www.docuseal.com) 打包上架到 TOS 7 应用中心的 Docker 应用（单容器、单数据卷、SQLite，零外部依赖）。

---

## What it does / 这是什么

DocuSeal lets you create reusable document templates (PDF or DOCX), send them out for signature, and track the whole signing process — all self-hosted on your TNAS.

- Web UI on host port `10010` → container `3000`
- Single SQLite database, stored on your NAS
- Runs as a non-root user, no privileged mode, no host networking
- Health endpoint: `/up`

## Package contents / 包内文件

```
docuseal.tar.gz
├── config.ini            # TOS application metadata
├── docuseal.lang         # 14 languages (zh-cn, zh-hk, en-us, ... pt-pt)
├── docuseal.svg          # application icon (SVG, transparent)
└── docker-compose.yml    # container definition
```

## docker-compose.yml

```yaml
version: "3.8"

services:
  docuseal:
    image: docuseal/docuseal:3.3.0
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "10010:3000"
    volumes:
      - ./data:/data/docuseal
    environment:
      - TZ=Asia/Shanghai
    healthcheck:
      test: ["CMD-SHELL", "(command -v curl >/dev/null && curl -fsS http://127.0.0.1:3000/up) || (command -v wget >/dev/null && wget -q -O /dev/null http://127.0.0.1:3000/up) || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

x-app-meta:
  web:
    port: 10010
    protocol: http
```

### Two deliberate details / 两处刻意的写法

**1. The volume is mounted at `/data/docuseal`, not `/data`.**
That path is the image's `WORKDIR`, and DocuSeal writes both its SQLite database and its generated secret file there. If the volume were mounted at `/data`, Docker would create `/data/docuseal` *as root* when the container is created, and the non-root container process (uid 1000) could not write to it — the container would crash-loop with `Permission denied @ rb_sysopen - /data/docuseal/docuseal.env`. Mounting the volume directly on that directory keeps it writable by the container user.

**2. The healthcheck uses `127.0.0.1` and `/up`, not `localhost` and `/`.**
Puma listens on IPv4 only, while `localhost` resolves to `::1` first inside this image (busybox `wget` picks the IPv6 address and gets `Connection refused`). `/up` is Rails' built-in health endpoint and returns `200` directly, instead of the `/` redirect.

## No credentials in this package / 包内零凭据

DocuSeal generates its own `SECRET_KEY_BASE` on first start and stores it in the data volume (`docuseal.env`, mode 600, owned by the container user). Every installation therefore gets a unique secret, and no credential is stored in this repository or in the package.

## Data location / 数据位置

The host side of `./data` is resolved by the platform to the application data root:

```
/Volume<N>/DockerAppData/docuseal/data
```

It contains `db.sqlite3` (documents, templates, submissions) and `docuseal.env` (generated secret). Data survives uninstall as long as "delete data" is not selected, and it also survives reinstall — the secret file is preserved, so existing sessions stay valid.

## Build

```bash
./build.sh            # produces docuseal.tar.gz + docuseal.tar.gz.sha256
```

## Release

- Tag: `1.0.0`
- Asset: `docuseal.tar.gz`
- Checksum: `docuseal.tar.gz.sha256`

## Compliance notes / 合规说明

- Image source: Docker Hub only (`docuseal/docuseal`), version tag pinned, never `:latest`.
- Runs as non-root (`user: "1000:1000"`); no `privileged`, no `network_mode: host`.
- No credentials or tokens are stored in this repository.
- `container_name` is intentionally omitted so Compose derives a globally unique name.
- The icon in this repository is original placeholder artwork created for this package; it is **not** the upstream project logo, to avoid trademark issues.
- This repository contains packaging metadata only. The application itself is developed by the DocuSeal project and distributed under its own license (AGPL-3.0). All credit belongs to the upstream authors.

## Links

- Upstream project: https://github.com/docusealco/docuseal
- Documentation: https://www.docuseal.com/docs
- Docker Hub image: https://hub.docker.com/r/docuseal/docuseal
- TOS 7 development guide: https://github.com/terramaster-tos/tos-app-pkg-tools
