# Aersia VIP Player — Self-Hosted Fork

A self-hosted fork of [fpgaminer/vip-html5-player](https://github.com/fpgaminer/vip-html5-player), an HTML5 player for the [Vidya Intarweb Playlist](https://www.vipvgm.net) curated by Cats777 at [Aersia.net](https://aersia.net).

Live instance: privately hosted.

---

## What's Different From Upstream

| Feature | Upstream | This Fork |
|---------|----------|-----------|
| Playback mode | Shuffle only | Shuffle + Sequential toggle |
| Session persistence | Volume + playlist | + Shuffle mode, last track, sequential position |
| PWA support | None | Installable, standalone display, custom icon |
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

- **Sequential mode** — play tracks in playlist order rather than random. Toggle via the shuffle button in the control bar (`fa-random` = shuffle, `fa-list-ol` = sequential).
- **Persistent state** — shuffle preference, last played track, and sequential position all survive page refresh and browser close.
- **PWA** — installable on Android (Chrome) and iOS (Safari) via Add to Home Screen. Launches full-screen with custom icon.
- **Privacy** — `robots.txt` + `noindex` meta tag. No inbound links. Not discoverable via search engines.
- **Live playlist** — roster JSON files served from a local Pi proxy (`/roster/`), refreshed weekly via cron from `vipvgm.net`. WAP and CPP still use XML from the original Aersia CDN. No local music storage required.

---

## Persistence Logic

State is stored in `localStorage` with the following keys:

| Key | Type | Purpose |
|-----|------|---------|
| `playlist` | string | Active playlist name (VIP, Mellow, etc.) |
| `shuffle` | boolean string | Playback mode |
| `last_track` | string | Track ID string — used to restore UI highlight and scroll position |
| `last_track_idx` | integer | Track array index — used to seed `g_previous` for correct sequential advance |

**Priority chain on load:** default → localStorage → URL hash (each overrides the previous).

**Sequential position fix:** the upstream player's `playNextTrack()` calculates the next index from `g_previous`, which is only populated during a session. On reload, `g_previous` is empty, causing sequential mode to always reset to track 0. This fork seeds `g_previous = [restoredIdx]` immediately after the restored track element is clicked during playlist load.

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
scp ./index.html pi:/var/www/aersia/
ssh pi 'sudo chown alex:www-data /var/www/aersia/index.html'
```

Nginx config at `/etc/nginx/sites-available/aersia` — HTTP redirects to HTTPS, SSL served from Let's Encrypt certificates.

---

## Playlists

VIP, Mellow, Exiled, and Source use the `vipvgm.net` JSON API. WAP and CPP still use XML from the original Aersia CDN.

| Playlist | Format | Source |
|----------|--------|--------|
| VIP | JSON | `vipvgm.net/roster.min.json` (local proxy) |
| Source | JSON | `vipvgm.net/roster.min.json` — filtered to tracks with `s_file`, served from `mu/source/` |
| Mellow | JSON | `vipvgm.net/roster-mellow.min.json` (local proxy) |
| Exiled | JSON | `vipvgm.net/roster-exiled.min.json` (local proxy) |
| WAP | XML | `wap.aersia.net/roster.xml` |
| CPP | XML | `cpp.aersia.net/roster.xml` |

**Why a local proxy?** `vipvgm.net` does not include `Access-Control-Allow-Origin` headers on its JSON roster files, blocking cross-origin requests from the browser. Audio files are served with permissive CORS headers and stream directly from `vipvgm.net/mu/`. The local proxy only applies to the small roster metadata files, not the audio itself.

**Roster refresh:** `update-aersia-roster.sh` runs weekly via cron and re-fetches the three JSON files to `/var/www/aersia/roster/`. If the fetch fails, the last known good copy continues to be served.

**Known limitation:** Some Source playlist tracks return 302 redirects because their `s_file` slug does not correspond to an existing file on `vipvgm.net`. This is an upstream data inconsistency in Cats777's roster, not a player bug.

---

## Credits

- Playlist curated by [Cats777](https://aersia.net) — all music © respective owners
- Original HTML5 player by [fpgaminer](https://github.com/fpgaminer/vip-html5-player)
- Fork by Alex Chuc

## License

MIT — see upstream repository.
