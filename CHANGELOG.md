# Changelog

All notable changes to the helper script, newest first. There are no tagged
releases: containers pick up `master` on their next `update`.

## Unreleased

### Fixed
- **Server unreachable after updating to upstream's November 2026 version**
  (2026-11-03). Upstream now listens on `localhost` unless `LISTEN_ADDRESS`
  is set (`::` only in its Docker image), so after `update` the readers and
  the health check could no longer connect. New installs write
  `LISTEN_ADDRESS=::` into the config, and `update` adds it once to config
  files that lack it (then restarts).

### Added
- **`CLAUDE.md`** (2026-11-03): working rules for this repo — `master` ships to every container, idempotency, the env file belongs to the user, testing on CT 112 before pushing.
- **Web app** (2026-11-03): upstream's CrossPoint Sync web app (`app/`) is
  built alongside the server (`npm ci && vite build`, the same web-only steps
  as upstream's Dockerfile) and served at `/app/`. Its `node_modules` is
  removed after the build to keep the 4 GB disk lean. Without it the server
  logged `serveStatic: root path 'app/dist' is not found`. Builds in about
  35 s within the default 512 MB.
