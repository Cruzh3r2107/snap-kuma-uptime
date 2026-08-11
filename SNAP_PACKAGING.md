# Snap Packaging — Uptime Kuma

This document describes how to build, install, test, and configure the
`vsingh-uptime-kuma` snap produced from this repository. All packaging lives under
`snap/` plus this file; no upstream source files were modified.

This is a **packaging repository**. It contains no upstream source — the build
clones Uptime Kuma from GitHub at a pinned tag. Nothing needs to be checked out
here to build.

- **Snap name:** `vsingh-uptime-kuma`
- **Version:** `2.5.0` (upstream tag `2.5.0`)
- **Base:** `core24`
- **Confinement:** `strict`
- **Plugin:** `nil` (custom `override-build`)
- **Service:** single daemon `vsingh-uptime-kuma` (runs `node server/server.js` via a wrapper)
- **Default port:** `3001` (overridable with `UPTIME_KUMA_PORT`)
- **Data directory:** `$SNAP_COMMON` (SQLite DB, uploads, screenshots, logs)

## 1. Prerequisites

Install snapcraft and a build backend (LXD is recommended):

```bash
sudo snap install snapcraft --classic
sudo snap install lxd
sudo lxd init --auto
sudo usermod -aG lxd "$USER"   # re-login for group change to take effect
```

## 2. Build

From the root of this repository:

```bash
export SNAPCRAFT_BUILD_INFO=1
snapcraft pack
```

This produces `vsingh-uptime-kuma_2.5.0_<arch>.snap` in the repository root.

What the build does:
0. Clones `louislam/uptime-kuma` at tag `2.5.0` (depth 1) into the part's
   source directory. The part previously used `source: .`, which built from
   whatever the local working copy was checked out at — in practice
   `origin/master`, several commits past the release, while still being
   labelled `2.5.0`. The tag makes the build reproducible and the version
   string honest.
1. Provisions the pinned Node.js runtime (20.19.4).
2. Runs `npm ci` (full install, incl. dev deps) — this also compiles the
   native `@louislam/sqlite3` addon via `node-gyp` (needs `build-essential`,
   `python3`, `libatomic1` at build time).
3. Runs `npm run build` (Vite) to produce the Vue frontend in `dist/`.
4. Runs `npm prune --omit=dev` to strip dev dependencies from `node_modules/`.
 5. Stages `bin/node`, `server/`, `dist/`, `db/`, `node_modules/`, `extra/`,
    `src/`, `package.json`, and the launcher wrapper `bin/uptime-kuma-wrapper`.
    `src/` is required because the backend imports shared code from it at
    runtime (e.g. `server/server.js` does `require("../src/util")`); without
    it the daemon crashes on startup with `Cannot find module '../src/util'`.

## 3. First test in devmode

Install without confinement first to confirm the service starts:

```bash
sudo snap install --devmode ./vsingh-uptime-kuma_2.5.0_*.snap
snap services vsingh-uptime-kuma
journalctl -u snap.vsingh-uptime-kuma.vsingh-uptime-kuma -f
```

Open the UI at <http://localhost:3001>.

## 4. Interfaces

| Interface         | Auto-connect | Purpose                                                              |
|-------------------|:------------:|---------------------------------------------------------------------|
| `network`         | yes          | Outbound connections to every monitored target and notifier         |
| `network-bind`    | yes          | Bind/listen on port 3001 for the web UI and websocket API           |
| `network-control` | **no**       | Raw ICMP sockets for `ping`-type monitors (connect only if needed)  |

`network` and `network-bind` auto-connect and cover all core functionality.

`network-control` is **manual**. Connect it only if ICMP `ping`-type monitors
fail under plain network confinement (this also depends on the host's
`net.ipv4.ping_group_range` sysctl):

```bash
sudo snap connect vsingh-uptime-kuma:network-control
```

## 5. Install with real (strict) confinement

Once devmode testing passes, install the locally built snap with strict
confinement:

```bash
sudo snap install --dangerous ./vsingh-uptime-kuma_2.5.0_*.snap
# Optional, only for ping-type monitors:
sudo snap connect vsingh-uptime-kuma:network-control
```

Verify:

```bash
snap connections vsingh-uptime-kuma
snap services vsingh-uptime-kuma
```

## 6. Configuration

Environment defaults are baked into the app definition:

- `NODE_ENV=production`
- `DATA_DIR=$SNAP_COMMON/` → data persists at `/var/snap/vsingh-uptime-kuma/common/`
- `UPTIME_KUMA_PORT=3001`

To change the port or host at runtime you can override via the app's own env
vars (`UPTIME_KUMA_PORT`, `UPTIME_KUMA_HOST`). External databases
(MariaDB/MySQL, PostgreSQL, MSSQL, MongoDB, Redis) are configured from within
the Uptime Kuma web UI; the snap ships with embedded SQLite by default.

## 7. Submitting to the Snap Store (optional)

```bash
snapcraft login
snapcraft register vsingh-uptime-kuma        # first time only, name must be available
snapcraft upload --release=stable ./vsingh-uptime-kuma_2.5.0_*.snap
```

## 8. Troubleshooting

- **Logs:** `journalctl -u snap.vsingh-uptime-kuma.vsingh-uptime-kuma -f`
- **Service state:** `snap services vsingh-uptime-kuma`
- **Interactive confined shell** (to reproduce AppArmor denials):
  `sudo snap run --shell vsingh-uptime-kuma.vsingh-uptime-kuma`
- **AppArmor/Seccomp denials:** watch `dmesg` / `journalctl -xe` for
  `apparmor="DENIED"` lines, or run `sudo snappy-debug` while exercising the app.
- **Ping monitors fail:** connect `network-control` (section 4) and confirm the
  bundled `ping` binary is reachable (`sudo snap run --shell vsingh-uptime-kuma.vsingh-uptime-kuma -c 'command -v ping'`).

## 9. Notes and caveats (from analysis)

- **Confinement is strict.** `network` + `network-bind` (both auto-connect)
  cover the core monitoring web app.
- **Port 3001** is >= 1024 and therefore bindable under strict confinement — no
  privileged-port workaround or install hook is required.
- **Data directory is relocatable** via `DATA_DIR`; set to `$SNAP_COMMON` so all
  state (kuma.db, uploads, screenshots, logs) is written to a snap-writable
  location. No layouts are required.
- **No layouts:** the only hardcoded absolute paths live in Docker-only
  `server/embedded-mariadb.js`, which the snap does not use (it defaults to
  embedded SQLite; use an external DB for MariaDB/MySQL/Postgres/etc.).
- **Native module** `@louislam/sqlite3` is compiled from source at build time
  (`build-essential` + `python3`); `libatomic1` is staged as a runtime dep.
- **`iputils-ping`** is staged so `@louislam/ping` can find a `ping` binary; the
  wrapper puts `$SNAP/usr/bin` on `PATH`.
- **`ca-certificates`** is staged so outbound TLS (HTTPS monitors, webhooks,
  OIDC) can validate certificates; the wrapper sets `NODE_EXTRA_CA_CERTS`.
- **Not covered by the default interface set** (intentionally left out):
  browser/real-browser monitors (Playwright + Chromium) and Cloudflare Tunnel
  (`node-cloudflared-tunnel` downloads/execs a binary at runtime, which strict
  confinement blocks). These are non-default features.
- **Node comes from the upstream nodejs.org tarball, not the Ubuntu archive.**
  This is deliberate and worth keeping. Ubuntu's `nodejs` package externalises
  some of Node's built-in JS modules to absolute paths under
  `/usr/share/nodejs`, which resolve outside `$SNAP` under strict confinement;
  Node then aborts at startup with `Cannot load externalized builtin` and
  SIGABRTs into a restart loop, with **zero AppArmor denials** to point at the
  cause. (The sibling `vsingh-sport-kiosk` snap hit exactly this and needs a
  `layout:` binding `/usr/share/nodejs` to work around it.) The upstream tarball
  embeds its builtins, so this snap is immune and needs no layout.
- Upstream ships an `AGENTS.md`/`CLAUDE.md` stating a pull-request contribution
  policy for `louislam/uptime-kuma`. It governs PRs to that project, not
  downstream packaging, and this repository neither vendors nor modifies any
  upstream source — the build clones a released tag and applies no patches.
