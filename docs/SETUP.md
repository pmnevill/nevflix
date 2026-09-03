# SETUP.md — Rebuild & Configuration Runbook

> **Purpose:** the ordered, app-by-app guide to bringing this system to its working state from scratch — the knowledge that would otherwise only live inside the app databases. This is a **knowledge-transfer** doc, not a restore script. For fast disaster-recovery of *this exact box*, use the physical-media backup of the config directories instead; use *this* doc to rebuild anywhere, to understand the configuration, or to onboard an AI assistant.
>
> Read `ARCHITECTURE.md` (the shape) and `GOTCHAS.md` (the traps) alongside this. This runbook references those rather than repeating them.

---

## Order of operations (why this order)

Deploy and configure in this sequence — each step depends on the previous ones existing:

1. **Host prep** — user/UID/GID, folder structure, permissions.
2. **gluetun stack** — VPN must be up before the containers that ride it.
3. **Prowlarr** — indexers must exist before the *arr apps can sync them.
4. **Trawl (+ Redis)** — the indexer proxy Prowlarr uses for Cloudflare-protected indexers.
5. **Radarr / Sonarr / Lidarr** — connect to Prowlarr + deluge.
6. **Soularr** — bridges Lidarr to slskd; needs Lidarr + slskd working first.
7. **Jellyfin / Navidrome / Jellyseerr** — streaming/consumption, point at the libraries.
8. **cloudflared** — remote access, last.

---

## 0. Host prep (do first — everything depends on it)

- **Create a single service user/group and use its UID/GID everywhere.** All acquisition/library containers must share it or imports fail silently (see `GOTCHAS.md` → UID/GID). Note the numeric UID:GID; every container below uses it.
- **Decide the storage layout up front** around the hardlink rule (see `GOTCHAS.md` → hardlinks): the download folder and each library that should hardlink must be **on the same filesystem and under a single shared parent mount**. The working layout here uses a narrow shared folder containing both `downloads/` and the movie library, mounted into the relevant apps as one mount.
- **Pre-create host folders and `chown` them to the service UID/GID** before starting containers (Docker auto-creates missing bind paths as `root:root`). Redis's data folder is the exception (UID 999).
- Concrete paths for *this* install are in `DEPLOYMENT.md`; on new hardware, choose paths that satisfy the same-filesystem hardlink rule.

## 1. gluetun stack (gluetun + deluge + slskd + resilio-sync)

This is one compose stack. gluetun owns the network namespace; the others attach via `network_mode: service:gluetun`.

- **gluetun**: set `VPN_SERVICE_PROVIDER`, `VPN_TYPE: wireguard`, `WIREGUARD_PRIVATE_KEY` (from `.env`), `WIREGUARD_ADDRESSES`, and a server location. Needs `cap_add: NET_ADMIN` and the tun device.
- **Publish ALL dependents' ports on gluetun**, not on the dependents (deluge web + torrent ports, slskd's ports, resilio's sync port). A dependent that declares its own `ports:`/`hostname:` will fail to be created (see `GOTCHAS.md`).
- **deluge**: after it's up, open its web UI and **enable the Label plugin** (Radarr/Sonarr need it for categories). Set its completed-download path to the shared downloads mount. Note: expect a possible **gluetun startup race** — if deluge shows no connectivity, restart the deluge container once the tunnel is healthy (see `GOTCHAS.md`).
- **slskd**: set Soulseek `username`/`password` (from `.env`/sanitized config), point its download/incomplete paths at the right mounts.
- **resilio-sync**: single-purpose; add the specific sync key via its web UI, destination on the appropriate mount.

## 2. Prowlarr (the indexer hub — configure before the *arr apps)

- Create the admin login.
- **Add indexers**: 1337x, LimeTorrents, Nyaa.si (adjust to taste).
- **Add the Cloudflare-bypass proxy**: Settings → Indexers → add a **FlareSolverr** proxy pointing at Trawl by **host IP + port** (not container name), and give it a **tag** (e.g. `flare`). Apply that tag only to indexers that need Cloudflare solving (1337x). Others bypass it. (See `GOTCHAS.md` → Trawl for why host-IP and for the "Test only proves reachability" caveat.)
- **Per-indexer "Apps Minimum Seeders"** can be set to reject low/zero-seeder releases before a grab (this is per-indexer, distinct from the sync-profile default, and distinct from "Seed Ratio").
- **Add the *arr apps as "Applications"** (Settings → Apps) once they exist (step 5), pasting each app's API key so Prowlarr syncs indexers down to them. You'll come back to this after the *arr apps are up.

## 3. Trawl (+ Redis) — Cloudflare/Turnstile solver

- Deploy `trawl` + `trawl-redis`. Trawl exposes a FlareSolverr-compatible API on its port.
- **Redis session cache**: Trawl reaches Redis by hostname. Because the platform forces plain-bridge networking (no name DNS), add a Docker **`links: [redis]`** entry on the Trawl service so the `redis` hostname resolves (see `GOTCHAS.md`). Verify the log shows `session cache connected`, not a Redis timeout.
- **Verify it actually solves** (not just that Prowlarr's Test is green): hit Trawl's API directly and confirm a real `cf_clearance` cookie + page HTML come back.
- Note: Trawl runs on normal networking, so its solving traffic exits the real WAN IP (not the VPN) — can draw a Cloudflare IP block under heavy retrying; route through gluetun if that recurs. Also mind the Prowlarr version caveat in `GOTCHAS.md`.

## 4. Radarr / Sonarr / Lidarr (the *arr apps)

For **each** app: create login, then:
- **Root folder**: point at that app's library path (movies / tvshows / music).
- **Download client**: add **deluge** (host IP + port; the password if set). Assign a category (e.g. radarr/sonarr/lidarr).
- **Media Management → enable "Use Hardlinks instead of Copy."** (Necessary but not sufficient — the *mount structure* must also allow hardlinks; see `GOTCHAS.md`. Verify with the `ln` test.)
- **Connect to Prowlarr**: copy the app's API key (Settings → General), then in Prowlarr add it under Applications (step 2) and sync — indexers appear in the app automatically. Don't add indexers manually per-app.
- **Backups**: point the app's backup dir at its `/backup` mount.

App-specific:
- **Sonarr**: because TV downloads and the TV library may be on **different filesystems** (they are here — see `DEPLOYMENT.md`), Sonarr will **copy** not hardlink. Enable **"Remove Completed"** on the deluge download client so the duplicate download is removed after successful import (transient, not permanent duplication).
- **Lidarr**: **keep "Allow Fingerprinting" ON** — it's the guard against Soularr importing wrong-artist music. Do **not** disable it to force imports (see `GOTCHAS.md`). Enable both its own indexer path (via Prowlarr) *and* the Soularr path (step 6) — they run independently.

## 5. Soularr (Lidarr ↔ slskd bridge)

- Soularr is a headless script (no UI) driven by its `config.ini` (see the sanitized copy in `configs/` for the full working settings).
- **Runs as root unless given Docker's `user:` directive** — set `user:` to the service UID/GID (this image ignores `PUID`/`PGID`; see `GOTCHAS.md`), so its move-based imports don't create `root:root` folders Lidarr can't clean up.
- Key `config.ini` settings that matter (full list in the sanitized config):
  - `[Lidarr]` / `[Slskd]` host URLs (host IP + port) and API keys (from `.env`/sanitized config).
  - **`album_prepend_artist = True`** and **`track_prepend_artist = True`** — critical to avoid wrong-artist grabs (see `GOTCHAS.md`).
  - **`minimum_filename_match_ratio`** (e.g. `0.5`+) — rejects loose/wrong-content matches.
  - `accepted_formats` / `accepted_countries` — tighten to reduce edition-mismatch failures.
  - **Section layout matters:** `album_prepend_artist` / `track_prepend_artist` / `minimum_filename_match_ratio` live under `[Search Settings]`; `rename_download_folders` lives under its own `[Download Settings]` section. Put each key under the right section header or Soularr ignores it. (The full working file is in `configs/soularr/config.ini`.)
- Expect some albums (obscure artists) to land in `failed_imports/` — that's Lidarr's guard working. Rescue via Lidarr **Interactive Import** pointed at the folder (see `GOTCHAS.md`).

## 6. Jellyfin / Navidrome / Jellyseerr

- **Jellyfin**: create the server + admin user; add libraries pointed at the movie and TV library mounts (and any extra library like a separate restoration-project folder). Enable hardware transcoding if the host supports it (pass the render device). Real-time monitoring picks up new files as the *arr apps import.
- **Navidrome**: point at the music library mount. Set env (`ND_DEFAULTTHEME`, etc.). Note: it reads **embedded tags** for grouping (see `GOTCHAS.md` → tags). Do **not** set `ND_FAVICON` (not a real option). Theme is per-browser.
- **Jellyseerr**: connect it to Jellyfin (URL + API key), set up request/approval as desired.

## 7. cloudflared (OPTIONAL — secondary remote access only)

- Primary access is on the local network (direct host IP + port); this step is only needed if you want off-network access. Skippable otherwise.
- Deploy the Cloudflare Tunnel container. In the Cloudflare Zero Trust dashboard, create the tunnel and add **public hostnames** mapping subdomains to each internal service (Jellyfin, Navidrome, Jellyseerr) by internal address + port.
- Set "Known Proxies"/trusted-proxy settings in the apps so they see real client IPs through the tunnel.
- Prefer subdomains over subpaths (fewer base-path headaches).

## 8. Post-setup verification checklist

- [ ] VPN is actually routing: a VPN-routed container shows the VPN's IP, not the home IP.
- [ ] A movie import **hardlinks** (same inode, link count ≥ 2 across download + library) — see `GOTCHAS.md` test.
- [ ] Trawl returns a real `cf_clearance` on a direct API call.
- [ ] Prowlarr indexers show up inside each *arr app after sync.
- [ ] A test grab flows end-to-end (search → deluge/slskd → import → appears in library → visible in Jellyfin/Navidrome).
- [ ] Lidarr fingerprinting is ON; Soularr `album_prepend_artist` is True.
- [ ] Remote access works from off-network with no VPN client.

---

## What this doc deliberately does NOT contain

- **Secrets** — those are in the local `.env` (git-ignored) and the physical-media backup, never here.
- **The SQLite databases** — intentionally not backed up to the repo (they carry machine-specific IPs/paths and don't port cleanly). This runbook is the substitute: reconfigure from it rather than restoring DBs. For same-box recovery, restore the physical-media backup instead.
- **Routine command syntax** — looked up in the moment; this doc captures decisions and ordering, not keystrokes.
