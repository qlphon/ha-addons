# FreeRADIUS + daloRADIUS

Single-container Home Assistant add-on bundling:

- **FreeRADIUS 3** — RADIUS auth (1812/udp) and accounting (1813/udp)
- **SQLite** — zero-admin local database (file lives in `/data/radius.db`)
- **daloRADIUS 2.2** — web UI (port 80, mapped to host 8080 by default)
- **Apache 2 + mod_php** — no PHP-FPM, so no FastCGI scoreboard / flock pain

See [DOCS.md](./DOCS.md) for installation, first login, and the full
rationale for each architectural choice (especially: why there is **no**
`apparmor.txt` in this folder).
