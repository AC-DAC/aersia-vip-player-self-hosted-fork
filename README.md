# Aersia VIP Player — Self-Hosted Fork

A self-hosted fork of [fpgaminer/vip-html5-player](https://github.com/fpgaminer/vip-html5-player), an HTML5 player for the [Vidya Intarweb Playlist](https://www.vipvgm.net) curated by Cats777 at [Aersia.net](https://aersia.net).

Live instance: privately hosted.

---

## What's Different From Upstream

| Feature | Upstream | This Fork |
|---------|----------|-----------|
| Playback mode | Shuffle only | Shuffle + Sequential toggle |
| Session persistence | Volume + playlist | + Shuffle mode, per-playlist last track and position |
| PWA support | None | Installable, standalone display, custom icon, lock screen controls |
| Hosting | GitHub Pages | Self-hosted on Raspberry Pi 4B via Nginx |
| Playlist format | XML (aersia.net CDN) | JSON (vipvgm.net) + local proxy on Pi |

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
- **Persistent state** — shuffle preference and last played track/position are stored per-playlist. Switching playlists restores your position in each independently.
- **PWA** — installable on Android (Chrome) and iOS (Safari) via Add to Home Screen. Launches full-screen with custom icon. Lock screen shows track title and artist with next/previous controls via the Media Session API.
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

**Sequential position:** `playTrack()` always pushes to `g_previous` and updates `g_previous_idx`, regardless of whether a track was reached via Next/Prev or by clicking directly. This means pressing Next always advances from the current track.

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

**Note on Cloudflare Access:** Zero Trust access control (email PIN gate) was evaluated but not implemented. Cloudflare Access intercepts requests before the service worker can authenticate, breaking PWA installation and offline capability. Since PWA support was a project requirement, Cloudflare Access was deprioritised in favour of obscurity-based privacy controls.

---

## Deployment

Copy updated files to the Pi:

```bash
scp ./index.html ./sw.js pi:/var/www/aersia/
ssh pi 'sudo chown alex:www-data /var/www/aersia/index.html /var/www/aersia/sw.js'
```

**Service worker versions:** v1 and v2 were written and deployed directly on the Pi (not in this repo). v3 is the first version tracked here — it caches the app shell (`/`, `index.html`, `manifest.json`, `favicon.ico`) and skips audio requests so they stream directly from the CDN. Bumping the `CACHE` constant forces clients to drop the old cache and install the new SW.

Nginx config at `/etc/nginx/sites-available/aersia` — HTTP redirects to HTTPS, SSL served from Let's Encrypt certificates.

---

## Playlists

| Playlist | Format | Status |
|----------|--------|--------|
| VIP | JSON | ✅ Active — `vipvgm.net/roster.min.json` (local proxy) |
| Mellow | JSON | ✅ Active — `vipvgm.net/roster-mellow.min.json` (local proxy) |
| Exiled | JSON | ✅ Active — `vipvgm.net/roster-exiled.min.json` (local proxy) |
| WAP | XML | ✅ Active — `wap.aersia.net/roster.xml` |
| CPP | XML | ✅ Active — `cpp.aersia.net/roster.xml` |
| Source | — | ❌ Disabled — see below |

**Why a local proxy?** `vipvgm.net` does not include `Access-Control-Allow-Origin` headers on its JSON roster files, blocking cross-origin requests from the browser. Audio files are served with permissive CORS headers and stream directly from `vipvgm.net/mu/`. The local proxy only applies to the small roster metadata files, not the audio itself.

**Roster refresh:** `update-aersia-roster.sh` runs weekly via cron and re-fetches the three JSON files to `/var/www/aersia/roster/`. If the fetch fails, the last known good copy continues to be served.

**Note on Source playlist:** The Source playlist (original/unremixed tracks) was implemented and tested but has been disabled due to upstream data inconsistency in Cats777's roster. The `roster.min.json` file uses an `s_file` field to reference source track slugs, but these slugs are inconsistently distributed across two CDN paths (`mu/` and `mu/source/`), with no field in the JSON to distinguish between them. The result is that the majority of Source tracks return 302 redirects regardless of which base URL is used. Since a mostly-broken playlist is worse than no playlist, Source has been removed from the player until Cats777's roster data is corrected.

---

## Credits

- Playlist curated by [Cats777](https://aersia.net) — all music © respective owners
- Original HTML5 player by [fpgaminer](https://github.com/fpgaminer/vip-html5-player)
- Fork by Alex Chuc

## License

MIT — see upstream repository.
