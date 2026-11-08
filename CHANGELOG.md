# Changelog

All notable user-visible changes to the Gitea TOS application package.

## 28.3.0-1 — 2026-11-09

### Added
- First release. Gitea (`gitea/gitea:28.3.0-rootless`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8026 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/giteadocker/`.
