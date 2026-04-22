# FreeRADIUS + daloRADIUS — Documentation

## What it does

Runs a full RADIUS stack inside one Home Assistant add-on:

- **FreeRADIUS 3** listens on `1812/udp` (auth) and `1813/udp` (accounting).
- **daloRADIUS** serves the admin UI on port `80` inside the container,
  published to `8080` on the HA host by default.
- **SQLite** keeps all state in `/data/radius.db`, which HA persists across
  restarts and updates.

## Installation

1. Add the repo (`https://github.com/qlphon/ha-addons`) to HA.
2. Install **FreeRADIUS + daloRADIUS** from the add-on store.
3. Configure options (see below) and start the add-on.
4. Open `http://<ha-host>:8080/` in a browser.
5. Log in with:
   - **username:** `administrator`
   - **password:** whatever you set in `daloradius_admin_password`

## Options

| Key | Default | Purpose |
|-----|---------|---------|
| `radius_shared_secret` | `testing123` | Shared secret for localhost NAS client. Change this. |
| `daloradius_admin_password` | `radius` | Web UI password for `administrator`. Change this. |
| `log_level` | `info` | `debug` switches FreeRADIUS to `-X` full-debug mode. |

## Why this design — and why there is **no** `apparmor.txt` here

### The original problem

When running this add-on under HA Supervisor, **PHP-FPM** failed at startup
with:

```
ERROR: failed to create lock. (Permission denied [13])
```

Crucially, it failed **only** under Supervisor. Plain `docker run` worked;
`docker run --security-opt apparmor=unconfined` worked; using Supervisor's
generated profile (e.g. `8eb5a3ed_freeradius`) failed — **even with the
profile widened to `/** rwix`, `capability,`, `mount,`, `unix,`, `signal,`,
`ptrace,`.**

### Root cause

AppArmor mediates `flock()`/`fcntl(F_SETLK)` through a separate permission
bit: **`k`** (lock). A rule like `/** rwix` permits read, write, execute
and mmap — but **not** file locking. Any call to `flock()` on a file the
profile only grants `rwix` to returns `EACCES` (errno 13). This has been
the case since Linux 4.4 introduced flock mediation in AppArmor.

PHP-FPM's scoreboard (in `sapi/fpm/fpm/fpm_scoreboard.c`) uses shared
memory plus a file lock for inter-worker synchronisation. No lock → no
startup.

### Why this add-on avoids it entirely

Two independent mitigations, belt-and-braces:

1. **No PHP-FPM at all.** We run Apache 2 with **`mod_php`**
   (`libapache2-mod-php`). PHP runs inside Apache's worker processes;
   there is no FastCGI scoreboard, no shared-memory pool, and no
   cross-process `flock()`. The problem surface disappears.

2. **AppArmor disabled for the add-on.** `config.yaml` has
   `apparmor: false`, **and this folder contains no `apparmor.txt`**.

   > The combination matters. Supervisor's rule (`supervisor/addons/model.py`)
   > is: if `apparmor.txt` is present in the add-on folder, the Supervisor
   > loads and applies it **regardless** of `apparmor: false`. The flag
   > only takes effect when no custom profile file exists. A leftover
   > `apparmor.txt` from earlier experiments is the #1 reason
   > `apparmor: false` appears not to work.

   With no profile file and the flag set to `false`, Supervisor launches
   the container with `--security-opt apparmor=unconfined`.

### If you ever want an AppArmor profile

If you later decide to re-enable AppArmor confinement, ship an
`apparmor.txt` alongside `config.yaml` that includes at minimum the
**`k`** flag on any path PHP or Apache might lock:

```
/** rwlkmix,
```

Note the `l` (link) and `k` (lock) flags — both are routinely missed in
hand-written profiles.

## Files / volumes

| Path | Persistence | Notes |
|------|-------------|-------|
| `/data/radius.db` | persistent (HA) | SQLite database, created on first start |
| `/data/options.json` | managed by HA | user-configured options |
| `/share` | persistent (HA) | available for backups / dumps if you need it |

## Troubleshooting

**Web UI shows a blank page or 500.** Check the add-on log —
`/proc/self/fd/2` is wired to the container's stderr so Apache errors
end up in the HA log viewer.

**`radtest` fails from localhost.** The shared secret in
`/etc/freeradius/3.0/clients.conf` is replaced at startup with your
`radius_shared_secret` option; make sure your `radtest` call uses the
same secret.

**`failed to create lock` reappears.** That means something re-introduced
PHP-FPM or an AppArmor profile. Confirm (a) you are on this Dockerfile
(which installs `libapache2-mod-php`, not `php-fpm`) and (b) there is no
`apparmor.txt` in the `freeradius/` add-on folder.

**Data survived an update but login fails.** `daloradius.conf.php` is
re-generated on every start from options; the operator row is created
once on first run. If you changed `daloradius_admin_password` after
first start, update the `operators.password` column manually (MD5 of
the new password).

## Support

Issues: <https://github.com/qlphon/ha-addons/issues>
