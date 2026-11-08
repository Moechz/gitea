# gitea-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Gitea](https://github.com/go-gitea/gitea).

## What this is

A four-file TOS Docker application archive (`giteadocker.tar.gz`) that installs
Gitea on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `giteadocker`)
- `giteadocker.lang` — 23-language superset (TOS requires 14)
- `giteadocker.svg` — application icon (1823 bytes, 2 SVG elements)
- `docker-compose.yml` — one service, image `gitea/gitea:28.3.0-rootless`

## How it works

| | |
|---|---|
| Container image | `gitea/gitea:28.3.0-rootless` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8026/` (container port 3000) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/etc/gitea` → `/Volume1/DockerAppData/giteadocker/etc`, `/var/lib/gitea` → `/Volume1/DockerAppData/giteadocker/data` |
| Logs | `docker logs giteadocker` (stdout/stderr) |

`etc` holds `app.ini`; `data` holds `gitea.db` (SQLite), repositories, avatars, attachments and the SSH host keys. Copying both folders is a complete backup.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 3000.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **Non-root.** Runs as `1000:1000`; the image is designed for the `git` user.
- **One volume, one web port.** `/data` keeps all state; only 3000 is published.
- **Health check** probes `/` on 3000.
- **Fixed image tag**, Docker Hub only, no privileged mode, no host network, no Docker socket.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `etc/app.ini` | Gitea configuration | first start | small | never overwritten; edit to set ROOT_URL/SSH |
| `data/gitea.db` | users, repositories metadata, issues and pull requests (SQLite) | first-run installer | grows with activity | never overwritten; back it up |
| `data/repositories` | Git repositories | when the first repository is created | grows with repositories | never overwritten; back it up |
| `/Volume1/DockerAppData/giteadocker/etc` | Gitea application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |
| `/Volume1/DockerAppData/giteadocker/data` | Gitea application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/giteadocker.tar.gz + .sha256
scripts/build.sh 28.3.0-2   # packaging iteration bump
scripts/build.sh 28.3.0-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/giteadocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v28.3.0-1`**, assets `giteadocker.tar.gz` +
   `giteadocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `giteadocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `28.3.0-1` → automated validation → review.

## License / branding

Gitea is distributed by its upstream authors under **MIT**. The container image is the official image from Docker Hub (`gitea/gitea:28.3.0-rootless`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

SSH clone (port 22) is not published. Use HTTP clone; if you need SSH, add a host port mapping and adjust `SSH_PORT`/`SSH_LISTEN_PORT` in `app.ini`.
After the first run, set `ROOT_URL` to your external NAS URL; otherwise clone and webhook links may point to `localhost:3000`.
The first-run installer is open until you create the administrator account. Complete it immediately after installation.
Plain HTTP on the LAN; do not expose the published port to the internet without a TLS reverse proxy.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
