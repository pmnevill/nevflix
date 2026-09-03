# CLAUDE.md — Orientation for AI Assistants (and humans in a hurry)

> You're looking at the configuration repo for a self-hosted media server. This file is the **map**. Read it first, then read the doc it points you to for whatever you're actually doing. Its job is to get you productive fast and to stop you from repeating mistakes that already cost hours to discover.

---

## What this system is (one paragraph)

A **ZimaOS** (CasaOS-based) home media server. It automatically acquires movies/TV (via torrents) and music (via Soulseek), streams them (Jellyfin for video, Navidrome for music), and routes all acquisition traffic through a **VPN kill-switch** so nothing leaks. Apps run as Docker Compose stacks managed by the ZimaOS dashboard; the real compose files live at `/DATA/.casaos/apps/<app>/docker-compose.yml`.

## Read in this order

1. **`ARCHITECTURE.md`** — the shape of the system and *why* it's built this way. Drive-agnostic. Start here to understand how the pieces fit.
2. **`GOTCHAS.md`** — the non-obvious constraints, traps, and dead-ends. **Read this before proposing any change.** Nearly every entry represents a wrong turn already taken; it will stop you from suggesting things that don't work.
3. **`DEPLOYMENT.md`** — this specific machine's hardware and concrete choices. The **ephemeral** layer: if you're helping rebuild on *different* hardware, treat this as "current state to understand," not "spec to copy."
4. **`compose/`** — the templated compose files (secrets replaced with `${VARS}`).
5. **`configs/`** — app config/database backups (disaster recovery).

## The load-bearing constraints — internalize these before touching anything

If you remember nothing else, remember these. Each has a fuller treatment in `GOTCHAS.md`.

- **All acquisition/library containers must share one UID/GID.** Mismatches cause *silent* import failures. Some images need `PUID`/`PGID`; some ignore it and need Docker's `user:` directive.
- **The download stack is VPN-isolated via `network_mode: service:gluetun`.** Dependents therefore **cannot** declare their own `ports:`/`hostname:` — those live on gluetun. A dependent that "won't appear" is usually violating this.
- **Hardlinks require the download folder and library folder to be on the same filesystem *and* under a single shared mount.** Two separate bind mounts to the same drive still break hardlinks silently (Radarr copies instead and logs nothing). This silently doubled disk usage before it was caught.
- **Don't disable Lidarr fingerprinting to force music imports.** It's the guard that catches Soulseek delivering the *wrong artist's* music. Disabling it trades "won't import" for "silently imports garbage."
- **Cross-service calls use the host LAN IP, not `localhost` or container names.** Don't "clean these up" — it breaks them. (This is also the main thing to fix when rebuilding on a new IP.)
- **ZimaOS rewrites compose files** and force-sets `network_mode: bridge`. Work around it; don't assume a hand-edit sticks.

## How to behave in this repo

- **Verify with ground truth; don't reason from assumptions.** The single most repeated lesson here: actual `docker inspect` / compose files / `ls -li` / logs beat confident theories. When diagnosing, pull the real state first. (Even the docs themselves contain a correction where a problem was misattributed before the real cause was found.)
- **A green "Test" button proves reachability, not correctness.** Prove the actual operation — a real hardlink, a real solve, a real import.
- **Labels lie.** A proxy named "Byparr" was running Trawl; a drive named "ssd-storage" is a spinning HDD. Check what's actually there.
- **Prefer decisions/constraints over procedures.** These docs deliberately explain *what to know to decide correctly*, and leave routine command syntax to be looked up. Keep it that way — don't bloat them with rediscoverable trivia.

## Secrets & safety

- The compose files here are **templated** — real credentials (VPN keys, API keys, Soulseek login) are replaced with `${VARS}` and kept in a local `.env` that is **not** in this repo. A full rebuild needs both this repo **and** that `.env` (stored separately/securely).
- App **databases** in `configs/` can still contain embedded secrets (e.g. indexer API keys) and machine-specific values (IPs, paths). Treat the repo as private regardless.

## The three-tier mental model (why this repo exists)

1. **Compose files** = what containers exist + how they're wired. (Here, templated.)
2. **App databases** = what was configured in the UIs. (Restorable, but carries this-machine specifics.)
3. **Integration knowledge** = the cross-app requirements and gotchas that live in *no* config file. **Captured in `GOTCHAS.md`** — and for a rebuild on new hardware, often more useful than the databases, since the DBs carry the old machine's baggage.

If you're an LLM helping stand up a *new* server: lean on `ARCHITECTURE.md` + `GOTCHAS.md` + `SETUP.md` and reconfigure fresh, rather than restoring databases full of the old machine's IPs and paths.
