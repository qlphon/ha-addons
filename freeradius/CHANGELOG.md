# Changelog

## 0.1.0 — 2026-04-22

Initial clean-slate rewrite.

- Base image: `debian:bookworm-slim` (no HA base, no s6-overlay).
- Web stack: Apache 2 + mod_php (no PHP-FPM, no FastCGI scoreboard).
- Database: SQLite at `/data/radius.db` (persistent per-add-on volume).
- daloRADIUS 2.2.2 installed from upstream tarball.
- FreeRADIUS SQL module pre-wired for SQLite.
- `apparmor: false` + **no `apparmor.txt`** so Supervisor runs the
  container `unconfined`.
- Single `run.sh` process manager with SIGTERM handling, launched under
  `tini`.
- Container listens on port 80 (mapped to host 8080) plus 1812/1813 UDP.
