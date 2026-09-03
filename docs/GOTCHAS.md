# GOTCHAS.md — Constraints & Hard-Won Lessons

> **Read this before changing anything.** These are non-obvious constraints and traps that cost real time to discover and that change *decisions*. This doc is deliberately about **why** and **what to know**, not step-by-step procedures — routine command syntax is left to be looked up in the moment. Hardware-specific facts live in `DEPLOYMENT.md`; this file is drive-agnostic and portable to a rebuild on different hardware.

---

## ZimaOS platform quirks

- **ZimaOS owns the compose files and will rewrite them.** The deployed compose files live at `/DATA/.casaos/apps/<app>/docker-compose.yml` (note: `/DATA` is a separate filesystem, so a `find / -xdev` misses them — search `/DATA/.casaos/apps` directly). The dashboard can silently overwrite hand-edits on its next save/redeploy. Verify a hand-edit survived, or make changes through the dashboard's YAML editor.
- **ZimaOS force-sets `network_mode: bridge` on services and won't let you remove it.** This has real consequences for container-to-container DNS (see "Networking"). Work *around* it rather than fighting it.
- The underlying OS is not a standard apt userland — expect some missing tools. Don't assume a given CLI is present.

## UID / GID — the #1 cause of silent import failures

- **Every container must run as the same UID/GID** (here, `1000:1001`). If the acquisition apps (Lidarr, Soularr, slskd, deluge, Radarr, Sonarr) don't agree, file-ownership mismatches make imports/moves fail — often silently or with a confusing mid-operation "permission denied."
- **LinuxServer.io images** honor `PUID`/`PGID`. Their PID 1 runs as root (the s6 supervisor — this is normal; `docker exec <c> id` showing root is expected), but the real app process runs as PUID/PGID. Check the *actual* process user with `docker top <container>`, not `docker exec ... id`.
- **Some images ignore `PUID`/`PGID`** and run as root regardless (observed: `mrusse08/soularr`). For these, use Docker's native `user: "1000:1001"` directive, which sets the UID before the entrypoint runs.
- **Deleting a file needs write permission on its *parent directory*, not the file** (Unix rule). A `root:root` folder blocks another UID from deleting files inside it even if the files are owned by that UID. Root cause of a real import failure here: a root-running app created `root:root` folders that the UID-1000 app then couldn't clean up after copying.
- **Pre-create and `chown` host folders to the right UID/GID before first container start**, especially with `create_host_path: true` (Docker creates missing paths as `root:root`). The `redis` official image is an exception — it runs as UID 999.

## Networking & VPN (gluetun)

### The gluetun kill-switch pattern (reusable)
- `gluetun` owns a network namespace; dependents (`deluge`, `slskd`, `resilio-sync`) attach via `network_mode: service:gluetun`, forcing **all** their traffic through the VPN. They have no network stack of their own to fall back to — so it's a kill-switch **by construction**, not a setting that can fail open. Reuse this for any container that must never leak.

### Consequences of `network_mode: service:gluetun`
- The dependent service **cannot declare its own `ports:`** (Docker errors: `conflicting options: port publishing and the container type network mode`). **Publish its ports on the gluetun service instead.** Same for `hostname:`. Symptom if violated: the container **never gets created** (compose rejects it), rather than crashing after start. So "a service silently doesn't appear" → check for `ports:`/`hostname:` on a `service:gluetun` service.

### gluetun startup races (known, not fully solved)
- Dependents can start querying the network **before gluetun's tunnel is healthy**, and then fail in misleading ways — stalled downloads, "no internet," connection errors that look like unrelated problems. Restarting the *dependent* container often clears it once the tunnel is up. `depends_on: gluetun: condition: service_healthy` helps but has proven not fully airtight. (A similar start-order race hit Trawl↔Redis.) When a VPN-routed container behaves as if it has no connectivity, suspect this before deeper theories.

### `localhost` ≠ the host inside a container
- Inside a container, `localhost` is *that container*. Cross-container references (e.g. Prowlarr→Lidarr) use the **host LAN IP**, which routes out and back to the other container's published port. **Do not rewrite these to `localhost` — it breaks them.** Container names (`http://lidarr:8686`) are cleaner but only work when the containers share a **user-defined** network with DNS. This hardcoded-IP dependency is the main **portability caveat** for rebuilding on different hardware (a changed host IP breaks these references).

### Default bridge network has no DNS
- Docker's **default** `bridge` network provides no container-name resolution — only user-defined networks do. If a service sets `network_mode: bridge` explicitly, Compose skips creating the project's named network and uses the plain bridge, so name-based references (`redis://redis:6379`) fail even with both containers up. This is why ZimaOS's forced `network_mode: bridge` broke Trawl↔Redis. Workaround: a legacy Docker `links:` entry writes a direct `/etc/hosts` entry regardless of network DNS.

### VPN provider note
- Current provider is **Mullvad** (WireGuard). Mullvad offers **no port forwarding** — this is only relevant if torrent *peer connectivity* ever proves genuinely limited (inbound peers behind NAT can't reach you). It was **not** the cause of the stalls seen here (that was the gluetun startup race above). If port forwarding ever becomes necessary, test an alternative provider via a **parallel throwaway gluetun instance** (a second gluetun + a throwaway client pointed at it), leaving the real stack untouched — never experiment on the working VPN container.

## Hardlinks — the big storage lesson

### The core rule
- **Hardlinks cannot cross physical filesystems.** They're two directory entries for the same on-disk data (same inode), valid only within one filesystem. For the *arr apps to hardlink instead of copy, **the download folder and the library folder must be on the same filesystem**.

### The Docker trap that silently broke it
- Even on the **same drive**, if downloads and library are mounted as **two separate bind mounts**, the container sees them as **different devices** and hardlinking fails with `Invalid cross-device link`. Radarr then **silently copies instead** (its method is literally `HardLinkOrCopy`, and it logs *nothing* about which path it took) — silently doubling disk usage on every import until noticed.
- **Fix:** make downloads and library **sub-paths of a single bind mount** (here: a narrow `library/` folder holding both `downloads/` and `movies/`, mounted as one `source → /data`). 
- **"Use Hardlinks instead of Copy" in the UI is necessary but not sufficient** — it does nothing if the mount structure defeats it. It was ON the whole time hardlinking was silently failing.
- **The definitive test** (independent of app logs) — attempt a hardlink directly in the container:
  ```
  docker exec radarr ln "/data/downloads/<file>" "/data/movies/test-hardlink"
  ```
  Silent success = works; `Invalid cross-device link` = still broken. To confirm a real import hardlinked, `ls -li` both copies — **same inode + link count ≥ 2** means one physical copy. (`du -sh` on two folders *separately* double-counts hardlinked files — verify with inodes, don't panic-diagnose from separate `du` runs.)

### Security: don't over-mount
- A shared parent mount exposes everything under it to the container. **Don't mount the whole app-data tree** into Radarr/Sonarr (that would expose VPN keys, other apps' configs). Use a **narrow** folder containing only downloads + the media library. Same-filesystem `mv` to relocate folders into it is instant.

### Cross-filesystem cases will always copy (accepted compromise)
- If a library and its downloads are deliberately on **different filesystems** (e.g. to balance capacity), imports there **always copy** — physics, not a bug. Minimize the cost by having the *arr app **remove the download after successful import** ("Remove Completed" on the download client), turning permanent duplication into brief transient duplication. Revisit true hardlinking only when storage layout changes.

### Moving a shared folder breaks every consumer
- A downloads/library folder may be mounted by **many** containers (here, up to five). Relocating it requires updating the `source:` in **every** service that mounts it, or the others silently point at an empty directory (a media server's library goes blank, an importer loses its input, etc.). Stop all affected containers, update every `source:`, recreate.

## Music pipeline (Lidarr + Soularr + slskd)

### Where the failures actually come from
- Flow: Lidarr's wanted list → **Soularr** text-searches Soulseek (via slskd) → downloads a pick → tells Lidarr to import. Lidarr *also* independently searches its own torrent indexers → deluge. **These two paths race**; it is *not* a Soulseek-first / torrent-fallback design.
- **slskd is a dumb pipe** — no concept of "correct artist." **Soularr's text-matching is the weak link:** a search can match and download a *completely different artist's* files (observed: a metalcore band's album returned for an unrelated artist's search), then file them under the name it *asked for*.
- **Lidarr is the safety net, not the problem.** Its acoustic-fingerprint / match-confidence check (`Album match is not close enough: X% vs 80%`) is what *catches* wrong downloads and quarantines them in `failed_imports/`. A large `failed_imports/` pile means the guard is **working**.

### Fixes (Soularr `config.ini`)
- **`album_prepend_artist = True`** (biggest single fix for wrong-artist grabs — puts the artist in the search query; confirmed by working community configs). Set `track_prepend_artist = True` too for consistency.
- **`minimum_filename_match_ratio`** (e.g. `0.5`+) — rejects downloads whose filename doesn't closely match the search. Purpose-built for the wrong-content problem.
- Tightening `accepted_formats`/`accepted_countries` reduces "Lidarr wants a specific edition Soulseek doesn't have" failures.
- **Watch the `config.ini` section headers:** these search keys go under `[Search Settings]`; `rename_download_folders` goes under `[Download Settings]`. A key under the wrong section is silently ignored.

### DO NOT disable fingerprinting to force imports
- Setting "Allow Fingerprinting = Never" (or forcing past the confidence rejection via Interactive Import) *will* push failures through — **but it removes the exact guard that catches wrong-artist downloads.** You trade "won't import" for "silently imports the wrong music." Keep fingerprinting ON; if legit obscure albums over-reject, lower the *confidence threshold* rather than disabling the check. (Observed consequence of disabling: wrong files landed in the library with coherent-but-wrong tags — a Navidrome album filed under the wrong artist. **Coherent-but-mismatched tags are the sign the *file* is wrong, not the tag.**)

### Rescuing failed_imports
- Files in `failed_imports/` are often the **only** copy (Soularr *moves* on import, so anything still there never made it to the library) — **don't bulk-delete without checking.** Soularr keeps a **denylist** so it won't auto-retry a parked album; rescue via Lidarr's **Interactive Import** pointed at the folder directly. `_1`-suffixed folders = the same album downloaded twice by the retry loop; verify which copy is complete before deleting either.

### Tags & Navidrome
- **Navidrome builds its library purely from embedded file tags** (ignores folder names and Lidarr's DB). Wrong embedded tags → wrong grouping. So a bad Soulseek upload's tags surface directly in Navidrome. Fixing requires editing the files' tags (any FLAC/tag tool does this) and rescanning — but first **distinguish wrong-file (different artist's music — delete & re-acquire) from right-music-bad-tags (retag).** Listen to confirm which.
- **`beets`** is the tool purpose-built to prevent this class of problem (auto-corrects tags against MusicBrainz at import). Worth adopting if obscure-music mistags become a recurring chore; composes alongside Lidarr rather than replacing it.
- **MusicBrainz treats collaborations as separate artists** ("X", "X & Family", "X/Y" are distinct entities), so they appear as separate artists — this is *correct* metadata, not a bug, and a folder-vs-tag mismatch there is legitimate. Lidarr has no real "merge artists" feature; fix *tags* for display grouping rather than fighting the backend.

## Navidrome specifics
- `ND_FAVICON` is **not a real config option** — it does nothing (was set here in error; remove it). **There is no supported way to replace the login logo** — the frontend (icon included) is compiled into the Go binary, so no bind mount can override it; only a source rebuild, or intercepting the asset request at the reverse-proxy layer (fragile — the hashed filename changes on upgrades). Options that *do* exist: `ND_INSTANCE_NAME`, `ND_UILOGINBACKGROUNDURL`, `ND_UIWELCOMEMESSAGE`. Theme is stored per-browser (localStorage); `ND_DEFAULTTHEME` only sets the fresh-browser default.

## Indexer proxy (Trawl, replacing Byparr)
- Prowlarr's proxy points at a **host IP + port**, not a container name — so swapping the container behind that port is transparent to Prowlarr (no reconfig on replacement). The proxy's display **name is cosmetic** (a proxy labeled "Byparr" was actually running Trawl — rename to avoid confusion).
- A green **"Test" only proves reachability, not that it's actually solving Cloudflare challenges.** Prove real behavior by hitting the solver's API directly and checking for a `cf_clearance` cookie + real page HTML in the response.
- Trawl↔Redis session caching broke due to ZimaOS's forced `network_mode: bridge` (no DNS for the `redis` hostname); fixed with a `links:` entry (see Networking). Do **not** reach for `network_mode: service:redis` — overkill, and it reintroduces the ports-move problem.
- **Prowlarr version caveat:** a recurring bug class shows "challenge solved" but the indexer still 403s (sometimes a Prowlarr-side header issue, not the solver's) — switching solvers may not fix it. This stack pinned an older Prowlarr to avoid a regression. Any FlareSolverr-style solver may also break periodically as Cloudflare changes.

## Disk-full events cause cascading, confusing failures
- 100%-full → **SQLite `disk I/O error` (code 10)** across apps — this signals "no space for the journal," not corruption. **Multiple unrelated apps failing this way at once points at the shared drive being full**, not each app. In-progress torrents get marked `errored` with partial data (Files tab shows allocated size, but `recheck` reports 0% because bytes don't match hashes — re-download rather than trust them). A DB caught mid-write *can* genuinely corrupt (distinct from the transient error) — after freeing space, confirm the app actually loads its data, not just that it stopped erroring.
- **Biggest space wins when full, in order:** (1) clear stalled `incomplete-downloads/` (disposable); (2) fix hardlink duplication and reclaim duplicated completed-download copies once imported; (3) relocate a library to another filesystem.

## General debugging principles that repeatedly paid off
- **Verify with ground truth; don't reason from assumptions.** Actual `docker inspect` / compose files / `ls -li` / logs repeatedly contradicted plausible theories. (Including in *this* project: a stall was misattributed to a VPN limitation when it was really a startup race — pulling the real cause beats a confident guess.)
- **A green "Test" proves reachability, not correctness.** Prove the actual operation (a real solve, a real hardlink, a real import).
- **Labels lie** — check what's actually there, not what it's named.
- **App logs on disk outlast the `docker logs` ring buffer** — grep the app's own rotated log files for historical failures.
- **Silent fallbacks are the worst traps** (copy-instead-of-hardlink, wrong-artist grabs — both silent). When something's mysteriously wrong, **test the specific operation in isolation** rather than theorizing.
