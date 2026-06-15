<h1 align="center">yagura 🏯</h1>

<p align="center">
  <strong>Drop-in, lightweight, Zabbix-compatible monitoring server.</strong><br>
  Reuse your existing Zabbix agents and templates — in a single static Go binary.
</p>

<p align="center">
  <a href="https://github.com/&lt;NArrow&gt;/yagura/releases"><img src="https://img.shields.io/github/v0.5beta/release/&lt;NArrow&gt;/yagura" alt="Release"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/&lt;NArrow&gt;/yagura" alt="License"></a>
  <img src="https://img.shields.io/badge/Go-static%20binary-00ADD8" alt="Go">
</p>

---

**yagura** (櫓) is a self-hosted monitoring server that speaks the **Zabbix agent protocol** and imports **Zabbix templates** — but ships as one small static binary you can drop onto an Alpine LXC and run in minutes. No PHP, no heavyweight database, no wall of widgets.

If you like the simplicity of modern tools like Beszel or Uptime Kuma but you've already invested in the Zabbix agent + template ecosystem, yagura lets you keep that ecosystem without running the full Zabbix stack.

## Features

- **Drop-in agent compatibility** — existing `zabbix-agentd` / `agent2` work unchanged. Just point `Server=` / `ServerActive=` at yagura.
- **Zabbix template import** — load Zabbix 6.x/7.x template JSON directly; items, triggers and macros are expanded automatically.
- **Single static binary** — `CGO_ENABLED=0`, musl-independent. One ~6 MB file, runs anywhere Linux runs.
- **Server-side host-down detection** — a silent host is flagged on its own, **no template trigger required** (see below). Works for passive, SNMP and active hosts.
- **Triggers** — a practical subset of Zabbix expression syntax: `last/avg/min/max/sum/count/nodata/change`, `and/or/not`, arithmetic, `K/M/G/T` and `s/m/h/d/w` suffixes.
- **SNMP v1/v2c polling** — pull metrics from routers, switches and PDUs alongside agents (stdlib implementation, no external deps).
- **Notifications** — Discord webhook / generic JSON webhook.
- **Bilingual UI (English / 日本語)** — auto-selected from the browser, switchable in the sidebar.
- **Tiny footprint** — built for homelabs, Raspberry Pi, and small VPS fleets. Backup = copy the data directory.

## Quick start (Alpine LXC)

Grab the latest static binary for your arch from [Releases](https://github.com/&lt;OWNER&gt;/yagura/releases) (`yagura-linux-amd64` / `yagura-linux-arm64`):

```sh
apk add --no-cache ca-certificates          # required for HTTPS webhooks (Discord, etc.)

# service user + group
addgroup -S yagura && adduser -S -D -H -G yagura -s /sbin/nologin yagura

# binary
install -m755 yagura-linux-amd64 /usr/local/bin/yagura

# OpenRC service (shipped in the repo)
install -m755 yagura.initd /etc/init.d/yagura
rc-update add yagura default
rc-service yagura start
```

The data directory (`/var/lib/yagura`) and log (`/var/log/yagura.log`) are created on first start. The web UI is now on **`:8086`**, and yagura listens for agent traffic (sender + active checks) on **`:10051`**.

### Enabling auth & overriding options

The init script sources `/etc/conf.d/yagura` if present:

```sh
# /etc/conf.d/yagura   (chmod 600 — it holds the token)
YAGURA_TOKEN="$(head -c24 /dev/urandom | base64)"
YAGURA_OPTS="-data /var/lib/yagura -http :8086 -trapper :10051"
```

Run `rc-service yagura restart` to apply. With a token set, the UI prompts for it once and API calls need `Authorization: Bearer <token>`.

## Configuration

| Flag       | Default  | Description                              |
|------------|----------|------------------------------------------|
| `-http`    | `:8086`  | Web UI / API listen address              |
| `-trapper` | `:10051` | Trapper (agent sender & active checks)   |
| `-data`    | `./data` | Data directory (`meta.json` + TSDB)      |
| `-token`   | (none)   | API bearer token; empty = no auth        |
| `-pollers` | `8`      | Passive-check worker count               |

`YAGURA_TOKEN` (environment) sets the token when `-token` is empty.

```sh
yagura -data /var/lib/yagura -http :8086 -trapper :10051
```

## Host-down detection

A monitored host is detected as down **without needing any template trigger**. yagura tracks the last time each host successfully returned data and, if a host goes silent past a threshold, raises an `Unreachable` problem on its own — shown on the dashboard, reflected in the host's status dot (green → red), and pushed to your webhook. It covers passive, SNMP and active hosts alike.

This matters because the stock *Zabbix Linux* template keys availability off the internal item `zabbix[host,agent,available]`, which yagura does not serve, and most SNMP templates ship no liveness item at all — so trigger-based down-detection would never fire. yagura handles it at the server level instead.

- Threshold = the **"Unreachable after"** setting (`unreachable_sec`, seconds).
- `0` = **auto**: 3× the host's fastest item interval, with a 120 s floor — tolerant of a couple of missed polls.
- Hosts that have never reported yet (just added / freshly imported) are not judged, to avoid startup false alarms. Recovery auto-resolves the problem.

## Language (i18n)

The UI auto-selects English or Japanese from the browser language (anything other than `ja` → English). Switch manually with the `EN` / `日本語` toggle at the bottom of the sidebar (the choice is stored in `localStorage`).

Translations live in `internal/web/static/i18n/{en,ja}.json` as a flat key → string map. To add a language, drop in `<lang>.json` with the same keys. Logs and server-generated problem names stay English as a common operational language. Locale files are embedded in the binary, so it remains a single file.

## Migrating from Zabbix

1. Point each agent at yagura — no reinstall, just edit `zabbix_agentd.conf`:
   ```ini
   Server=<yagura-ip>          # allow passive checks
   ServerActive=<yagura-ip>    # active checks → :10051
   Hostname=Hogehoge              # must match the host name in yagura (active checks & sender)
   ```
2. In Zabbix, export the templates you use as **JSON** (`Data collection → Templates → Export`).
3. Import the JSON in yagura's **Templates** page, then link it when you add a host — items, triggers and the `/Template/` host refs in trigger expressions are rewritten to the host name automatically.

> Convert YAML exports first: `yq -o=json template.yaml > template.json`
> `zabbix_sender -z <yagura-ip> -s web01 -k custom.metric -o 42.5` works too — create the receiving item as type `trapper`.

## SNMP

Set a host's community / port (default `public` / `161`) and the SNMP GET items from your template start collecting immediately. Counters like `ifHCInOctets` get Change-per-second + Multiplier preprocessing applied, so bits/s is recorded directly (first sample and counter resets are dropped). `walk[...]` OIDs, LLD-derived items and SNMPv3 are not supported (imported disabled — rewrite to a direct-OID GET item if you need them).

## Status & limitations

yagura targets **agent + template compatibility**, not full Zabbix parity. Built in: passive / active / trapper collection, SNMP v1/v2c GET, template JSON import, the common trigger functions, and server-side host-down detection. **Not** implemented:

- Low-Level Discovery (LLD)
- Most of the preprocessing pipeline (Change-per-second & Multiplier are supported)
- SNMPv3, `walk[]`, and CALC / DEPENDENT / HTTP-agent items
- JSON-RPC API compatibility
- User macros `{$MACRO}` (item interval falls back to 60 s)

Unsupported items import in a **disabled** state with a note, so nothing silently breaks — enable and drive them via trapper (cron + `zabbix_sender`) if needed.

## HTTP API

All JSON. With `-token` set, send `Authorization: Bearer <token>`.

```
GET  /api/overview                  host / problem counts, worst severity
GET  /api/hosts                     · POST {name, addr, templates:[]}
POST /api/hosts/{id}/link           · POST /api/hosts/{id}/unlink   {template}
GET  /api/items?host={id}           · POST {hostid, key, delay, value_type}
GET  /api/history?item={id}&from=&to=
GET  /api/problems                  · GET /api/triggers
POST /api/templates/import          (paste a Zabbix JSON export)
GET|PUT /api/settings               · POST /api/test-alert
```

## Build from source

```sh
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o yagura .
```

`CGO_ENABLED=0` is what makes the binary fully static (musl-independent), so it runs as-is on Alpine.

## Testing

```sh
go test ./...
go build -o bin/faker ./cmd/faker
bin/faker agent &                     # fake zabbix-agent on :10050
bin/faker send <host> <key> <value>   # zabbix_sender equivalent
```

## Contributing

Issues and PRs welcome — translations especially: copy `internal/web/static/i18n/en.json` to `<lang>.json` and translate the values.

## License

[MIT](LICENSE) © &lt;YOUR NAME&gt;

---

> **Not affiliated with, endorsed by, or sponsored by Zabbix SIA.**
> "Zabbix" is a trademark of Zabbix SIA. yagura is an independent, clean-room implementation built against the publicly documented agent protocol and template format. It contains no Zabbix source code.
# yagura
# yagura
# yagura
# yagura
# yagura
# yagura
# yagura
