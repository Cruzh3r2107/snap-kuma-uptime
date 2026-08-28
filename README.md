# snap-kuma-uptime

Uptime Kuma is an open-source, free and easy-to-use self-hosted monitoring tool. Uptime Kuma is compatible with multiple platforms including Linux, Windows 10 (x64) and Windows Server.  Monitoring uptime has never been easier and Uptime Kuma offers exactly this, with a simple but effective and powerful dashboard.

This repository packages it as the `vsingh-uptime-kuma` snap, built from
upstream `louislam/uptime-kuma` at tag `2.5.0`.

## Install

```bash
sudo snap install --dangerous ./vsingh-uptime-kuma_2.5.0_amd64.snap
```

## Use it

Browse to <http://localhost:3002> and create your admin account on first visit.
Everything else — monitors, notifications, status pages — is configured from
the web UI.

```bash
snap services vsingh-uptime-kuma
snap logs vsingh-uptime-kuma -f
```

## Ping monitors need one extra step

HTTP(s), TCP, DNS, gRPC, MQTT and database monitors work out of the box.

**ICMP ping monitors do not.** They need a raw socket, which strict confinement
denies until you connect this interface by hand:

```bash
sudo snap connect vsingh-uptime-kuma:network-control
```

Without it, ping-type monitors fail while every other monitor type keeps
working. This is deliberate: raw socket access is broad, so it is opt-in.

## Your data

Everything lives in `/var/snap/vsingh-uptime-kuma/common/`:

```
kuma.db          SQLite database: monitors, history, users, settings
upload/          uploaded icons and images
screenshots/
```

This is `$SNAP_COMMON`, so it survives refreshes and reverts. Back up
`kuma.db` and you have your entire monitoring setup.

An external MariaDB/MySQL, PostgreSQL, MSSQL, MongoDB or Redis backend can be
configured from the web UI instead of the embedded SQLite.

## Troubleshooting

The web UI binds all interfaces on port 3002, so it is reachable from your LAN.
Put it behind a reverse proxy or a VPN if that is not what you want.

Two features are intentionally unavailable, because both need to run or fetch
binaries that strict confinement blocks:

- real-browser monitors (Playwright + Chromium)
- Cloudflare Tunnel

## Credits

Uptime Kuma is created and maintained by
[Louis Lam](https://github.com/louislam/uptime-kuma) and its contributors. All
of the software this snap runs is theirs; this repository only packages it.

Licensed MIT, the same as upstream.

**This is an unofficial package.** It is not produced or endorsed by the Uptime
Kuma project. Report packaging problems to
[this repository's issues](https://github.com/Cruzh3r2107/snap-kuma-uptime/issues),
not to upstream — and please confirm a bug is not packaging-specific before
raising it with them.

---

Build instructions and packaging internals are in
[SNAP_PACKAGING.md](SNAP_PACKAGING.md).
