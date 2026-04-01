# Aersia VIPVGM Player — Self-Hosted Fork

A self-hosted fork of [fpgaminer/vip-html5-player](https://github.com/fpgaminer/vip-html5-player), an HTML5 player for the [Vidya Intarweb Playlist](https://www.vipvgm.net) curated by Cats777 at [Aersia.net](https://aersia.net).

Live instance: privately hosted.

---

## What's Different From Upstream

| Feature | Upstream | This Fork |
|---------|----------|-----------|
| Playback mode | Shuffle only | Shuffle + Sequential toggle |
| Previous button | Goes to previous track | Shuffle: restarts current track. Sequential: goes to previous track |
| Session persistence | Volume + playlist | + Shuffle mode, per-playlist last track and position |
| Hosting | GitHub Pages | Self-hosted on Raspberry Pi 4B via Nginx |
| Playlist format | XML (aersia.net CDN) | JSON (vipvgm.net) + local proxy on Pi |
| Playlists | VIP, Mellow, Source, Exiled, WAP, CPP | VIP, Source, Mellow, Exiled, Omni, WAP, CPP |

---

## Infrastructure

```
Browser → Cloudflare (TLS proxy) → Home router (port forward) → Nginx on Pi → index.html
```

| Component | Detail |
|-----------|--------|
| Server | Raspberry Pi 4 Model B 4GB, Debian Trixie ARM64 |
| Web server | Nginx 1.26 |
| TLS | Let's Encrypt via Certbot (DNS-01 challenge, Cloudflare plugin) |
| DNS | Cloudflare — proxied, home IP hidden |
| Dynamic DNS | Shell script + cron, Cloudflare API, runs every 5 minutes |
| Domain | Subdomain isolates from future portfolio at root domain |

---

## Features

- **Sequential mode** — play tracks in playlist order rather than random. Toggle via the shuffle button in the control bar (`fa-random` = shuffle, `fa-list-ol` = sequential). Pressing Next always advances from the currently playing track, regardless of whether it was reached via the next button or by clicking directly.
- **Previous button behaviour** — in shuffle mode, Previous restarts the current track from the beginning. In sequential mode, Previous navigates to the prior track in the playlist.
- **Persistent state** — shuffle preference and last played track/position are stored per-playlist. Switching playlists restores your position in each independently.
- **Source fallback** — Source playlist tracks attempt to stream from `mu/source/` first. On failure, the player automatically retries from `mu/` before skipping to the next track.
- **Omni playlist** — merges VIP, Mellow, and Exiled into a single A-Z sorted playlist. Fetches all three rosters in parallel and concatenates results before rendering the track list.
- **Privacy** — `robots.txt` + `noindex` meta tag. No inbound links. Not discoverable via search engines.
- **Live playlist** — roster JSON files served from a local Pi proxy (`/roster/`), refreshed weekly via cron from `vipvgm.net`. WAP and CPP still use XML from the original Aersia CDN. No local music storage required.

---

## Persistence Logic

State is stored in `localStorage` with the following keys:

| Key | Type | Purpose |
|-----|------|---------|
| `playlist` | string | Active playlist name (VIP, Mellow, etc.) |
| `shuffle` | boolean string | Playback mode |
| `last_track_<playlist>` | string | Track ID string per playlist — restores UI highlight and scroll position |
| `last_track_idx_<playlist>` | integer | Track array index per playlist |

**Priority chain on load:** default → localStorage → URL hash (each overrides the previous).

**Per-playlist position:** each playlist stores its last position independently. Switching playlists restores where you left off in the target playlist rather than starting from the top.

**Sequential position:** `playTrack()` only pushes to `g_previous` when the incoming `trackid` differs from `g_previous[g_previous_idx]`. This prevents double-push when called from `playNextTrack()`, which already pushes before calling `playTrack()`. Direct clicks from the track list still record correctly.

---

## Dynamic DNS

`update-cloudflare-dns.sh` runs every 5 minutes via cron:

1. Fetches current public IP from `ifconfig.me`
2. Fetches current A record value from Cloudflare API
3. If different, updates the A record via PUT
4. Logs all activity to `/var/log/cloudflare-dns-update.log`

Requires a Cloudflare API token scoped to `Zone → DNS → Edit + Read` for the target zone.

---

## Access & Privacy

The live instance is privately hosted. The site is intentionally undiscoverable via:
- `robots.txt` disallowing all crawlers
- `noindex, nofollow` meta tag
- No public inbound links
- Cloudflare proxy hiding the origin IP

**Note on Cloudflare Access:** Zero Trust access control (email PIN gate) was evaluated but not implemented. Cloudflare Access would block requests before the browser can authenticate, incompatible with how the player fetches roster files. Deprioritised in favour of obscurity-based privacy controls.

**Note on PWA:** A PWA manifest and service worker were implemented and tested. Background audio playback on locked screens proved unreliable across Android (including GrapheneOS) and iOS due to OS-level power management killing the browser process. Since a broken PWA is worse than no PWA, it was removed. A companion native app is planned as the correct solution for background playback and lock screen controls.

---

## Deployment

Copy updated files to the Pi:

```bash
scp ./index.html pi:/var/www/aersia/
ssh pi 'sudo chown alex:www-data /var/www/aersia/index.html'
```

Nginx config at `/etc/nginx/sites-available/aersia` — HTTP redirects to HTTPS, SSL served from Let's Encrypt certificates.

---

## Playlists

| Playlist | Format | Status |
|----------|--------|--------|
| VIP | JSON | ✅ Active — `vipvgm.net/roster.min.json` (local proxy) |
| Source | JSON | ✅ Active — `vipvgm.net/roster.min.json`, filtered to `s_file` tracks with `mu/source/` → `mu/` fallback |
| Mellow | JSON | ✅ Active — `vipvgm.net/roster-mellow.min.json` (local proxy) |
| Exiled | JSON | ✅ Active — `vipvgm.net/roster-exiled.min.json` (local proxy) |
| Omni | JSON | ✅ Active — VIP + Mellow + Exiled merged and sorted A-Z |
| WAP | XML | ✅ Active — `wap.aersia.net/roster.xml` |
| CPP | XML | ✅ Active — `cpp.aersia.net/roster.xml` |

**Why a local proxy?** `vipvgm.net` does not include `Access-Control-Allow-Origin` headers on its JSON roster files, blocking cross-origin requests from the browser. Audio files are served with permissive CORS headers and stream directly from `vipvgm.net/mu/`. The local proxy only applies to the small roster metadata files, not the audio itself.

**Roster refresh:** `update-aersia-roster.sh` runs weekly via cron and re-fetches the three JSON files to `/var/www/aersia/roster/`. If the fetch fails, the last known good copy continues to be served.

**Source playlist:** VIP contains arranged/remixed versions of tracks. Source contains the original unremixed versions of the same tracks. Both playlists contain the same 2323 tracks. Source tracks attempt `mu/source/` first and fall back to `mu/` automatically — no tracks are skipped due to CDN path inconsistency.

**Omni playlist:** Client-side merge of VIP, Mellow, and Exiled. Three roster files are fetched in parallel via `$.when.apply`, each parsed with its own config, concatenated, then sorted A-Z by game + title before the track list is rendered.

---

## Credits

- Playlist curated by [Cats777](https://aersia.net) — all music © respective owners
- Original HTML5 player by [fpgaminer](https://github.com/fpgaminer/vip-html5-player)
- Fork by Alex Chuc

## License

MIT — see upstream repository.
