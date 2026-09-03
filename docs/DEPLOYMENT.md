# DEPLOYMENT.md — This Install's Specific Facts

> **This file is the ephemeral, hardware-specific layer.** Everything here is true of *this particular machine* and would be **replaced** when standing up a new server. The architecture (`ARCHITECTURE.md`) and constraints (`GOTCHAS.md`) are drive-agnostic and portable; this file records the concrete choices *this* deployment happens to make. If you're rebuilding on different hardware, read this only to understand the current setup — don't copy its specifics.

---

## Hardware

- **Host:** Dell Latitude E7450 laptop running ZimaOS (CasaOS-based).
- **Internal drive — NVMe SSD (fast):** holds the OS and `/DATA` (the ZimaOS data partition, ~222 GB, ~204 GB free). Physically fixed as the internal drive. `/media/ZimaOS-HD` is a **symlink to `/DATA`** (same drive, two paths).
- **External drive — WD2500BEVT, 2.5" 5400 RPM HDD (slow), over USB:** this is `/media/ssd-storage` (`/dev/sdb1`, ~229 GB). **The name "ssd-storage" is a misnomer — it is a spinning HDD, not an SSD.**
- The laptop has **no room for a second internal drive** (internal bay holds the NVMe; the M.2 slot can't fit a 2.5"/3.5" drive). Any capacity expansion must be **external USB**. It has 3 USB-A ports (all USB 3.0), one used by the external drive.

## Current drive/workload situation (and a known inefficiency)

- **The busy, I/O-heavy data is on the *slow* USB HDD** — downloads, the movie/music libraries, and most app config all live under `/media/ssd-storage/docker/...`. The fast NVMe mostly holds the OS and some app data under `/DATA/AppData`.
- **Why it's like this:** the original setup followed a YouTube guide that assumed a single large storage array, so "put the whole Docker data root on the array" made sense *there*. Copied onto this two-drive machine, it landed the heavy workload on the slow drive — the opposite of ideal. There is **no deliberate benefit** to the Docker root being on the external HDD here; it's an inherited convention.
- **Known optimization (not yet done, no new hardware needed):** relocate hot, small, random-I/O workloads onto the NVMe — incomplete-downloads staging, any transcode cache, and the SQLite-heavy app databases — while keeping **bulk media on the USB HDD**. The NVMe is only ~204 GB free, so it's for hot/small data, never the bulk library. This is a deployment tuning task to revisit when there's appetite; the architecture doesn't depend on it.

## Where things physically live right now

- **App config / databases** are split across two roots (a quirk of how apps were added over time):
  - `/DATA/AppData/` → radarr, sonarr, jellyfin, jellyseerr (+ some system apps)
  - `/media/ssd-storage/docker/arrs/` → lidarr, prowlarr, soularr
  - `/media/ssd-storage/docker/` → navidrome, gluetunstack (deluge/slskd/gluetun/resilio configs), trawl, etc.
- **Media libraries:**
  - Movies: `/media/ssd-storage/library/movies` (USB HDD)
  - Music: `/media/ssd-storage/docker/media/music` (USB HDD)
  - TV: `/media/ZimaOS-HD/media/tvshows` i.e. `/DATA/media/tvshows` (**NVMe** — moved here; see below)
- **Downloads:**
  - Deluge completed → `/media/ssd-storage/library/downloads` (USB HDD)
  - slskd → `/media/ssd-storage/docker/downloads/slskd` (USB HDD)
  - incomplete-downloads → `/media/ssd-storage/docker/incomplete-downloads/*` (USB HDD)

## The two deployment-specific compromises to know

1. **Movies hardlink; TV copies.** Movies + their downloads are both on the USB HDD under a shared `library/` mount, so Radarr **hardlinks** (no duplication). The **TV library was deliberately moved to `/DATA` (NVMe)** to relieve HDD space pressure (the ~140 GB TV library wouldn't fit in the HDD's free space). Because TV downloads (HDD) and the TV library (NVMe) are now on **different filesystems**, Sonarr **always copies** TV imports — unavoidable while they're split. Mitigation in place: Sonarr should remove the download after successful import so the duplicate is transient, not permanent. Revisit true TV hardlinking only when storage capacity increases.

2. **Cross-container references use the host IP `192.168.1.103`.** App-to-app links (Prowlarr↔the *arr apps, download clients, Trawl proxy) are configured with this literal LAN IP, not container names. **On a rebuild with a different host IP, these all need updating.** (See the `localhost`/DNS notes in `GOTCHAS.md` for why it's an IP and not `localhost`.)

## Remote access
- **Cloudflare Tunnel** (`wisdomsky/cloudflared-web`, host network) is the primary remote-access path — public HTTPS subdomains for Jellyfin / Navidrome / Jellyseerr, no client needed on the connecting device. (Tailscale was previously also running but was removed.)

## Cleanup / removed / parked (verified against live state)
- **Removed:** `byparr` (replaced by `trawl`). Stray `qbittorrent` and `big-bear-gluetun` remnants were also cleaned up (may leave empty `AppData` folders behind — harmless).
- **`tailscale` is still present and running, but parked / under evaluation** — it is NOT part of the current architecture. Remote access is handled by the Cloudflare Tunnel. Tailscale is kept only as a possible future remote-access option and can be ignored for the purposes of this repo. (Don't document it as load-bearing; it isn't.)
- **`ND_FAVICON` was removed** from Navidrome (it was never a real Navidrome option — see `GOTCHAS.md`).

## Known cosmetic discrepancies (not bugs — documented so they don't confuse)
- **Prowlarr version label vs. actual image.** Prowlarr's `image:` is pinned to `version-2.4.0.5397` (a deliberate rollback to avoid a 2.5.2 regression) — **this is what actually runs and is authoritative.** ZimaOS's dashboard metadata (`x-casaos: version:`) may still *display* `2.5.2`; that label is stale and has no functional effect. Trust the `image:` line, not the dashboard label. (Classic "labels lie" — see `GOTCHAS.md`.)
