# homelab-config

Configuration reference and knowledge base for a self-hosted **ZimaOS** media server (Jellyfin + the *arr stack + Soulseek, VPN-isolated downloads, Cloudflare Tunnel remote access).

> **What this repo is — and isn't.**
> This is a **knowledge-transfer and reference** repo, not a restore mechanism. It exists so that (a) a human or an AI assistant can understand the system quickly and rebuild it without repeating the integration pain it took to get right, and (b) the compose files and key configs are captured in a readable, version-controlled form.
> **Fast disaster recovery is handled separately** by a full config backup on physical media — *not* by this repo. The app databases (with their machine-specific IPs, paths, and embedded secrets) are deliberately **not** committed here; `SETUP.md` is the substitute (reconfigure from it rather than restoring a database).

---

## Start here

- **AI assistants:** read [`CLAUDE.md`](./CLAUDE.md) first — it's the orientation map and lists the load-bearing constraints to internalize before suggesting any change.
- **Humans:** skim [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) for the shape, then [`docs/GOTCHAS.md`](./docs/GOTCHAS.md) before touching anything.

## What's here

| Path | What it is |
|---|---|
| [`CLAUDE.md`](./CLAUDE.md) | Orientation entry point + load-bearing constraints. Read first. |
| [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) | The system's shape and *why* it's built this way. **Drive-agnostic** — portable to a rebuild on any hardware. |
| [`docs/GOTCHAS.md`](./docs/GOTCHAS.md) | Non-obvious constraints, traps, and dead-ends. Every entry is a wrong turn already taken. |
| [`docs/DEPLOYMENT.md`](./docs/DEPLOYMENT.md) | *This machine's* concrete specifics (hardware, which path is which drive, current compromises). The **ephemeral** layer a new deployment replaces. |
| [`docs/SETUP.md`](./docs/SETUP.md) | Ordered, app-by-app rebuild/configuration runbook. |
| [`compose/`](./compose/) | Templated Docker Compose files (secrets → `${VARS}`). Reference copies — ZimaOS manages its own canonical versions. |
| [`configs/`](./configs/) | Sanitized plain-text app configs (Soularr `config.ini`, slskd `slskd.yml`), secrets → `__PLACEHOLDER__`. |
| [`.env.example`](./.env.example) | The full secret inventory as a template. Copy to `.env` and fill in real values. |
| [`.gitignore`](./.gitignore) | Guarantees `.env`, databases, logs, and cache never get committed. |

## Secrets

- **No real secrets live in this repo.** Compose secrets are templated to `${VARS}`; config-file secrets are `__PLACEHOLDER__`. Real values live only in a local `.env` (git-ignored) and your password manager / physical backup.
- **A full rebuild needs this repo *plus* a populated `.env`.** See [`.env.example`](./.env.example) for the complete list of what's needed and where each value comes from.
- App databases (Prowlarr/Radarr/Sonarr/etc.) contain their own embedded secrets and are **not** in this repo — they're in the separate physical-media backup. On a from-scratch rebuild, each app generates fresh API keys on first run.
- **Keep this repo private regardless** — the sanitized configs and compose still reveal the full architecture.

## The three-tier model (why the repo is scoped the way it is)

1. **Compose files** — what containers exist + how they're wired. *(Here, templated.)*
2. **App databases** — what was configured in the UIs. *(NOT here — physical backup for same-box restore; `SETUP.md` for rebuild-anywhere.)*
3. **Integration knowledge** — the cross-app requirements and gotchas in no config file. *(Captured in `docs/GOTCHAS.md` + `docs/SETUP.md` — the most valuable part for a rebuild on new hardware.)*

## Using this to rebuild

1. Read `CLAUDE.md` → `docs/ARCHITECTURE.md` → `docs/GOTCHAS.md`.
2. Follow `docs/SETUP.md` in order (host prep → gluetun stack → Prowlarr → Trawl → *arr apps → Soularr → streaming → cloudflared).
3. Reference `compose/` for the container definitions and `configs/` for the app config settings.
4. Populate secrets from your `.env` / password manager as you go.
5. Consult `docs/DEPLOYMENT.md` only to understand the *current* machine — replace its specifics with your own hardware's.
