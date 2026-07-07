# Kwiat Uczuć — Claude Instructions

## Project Overview

**Kwiat Uczuć** is an evening emotional ritual PWA for families. Single-page app, single HTML file, no build step.

**Live app:** https://kwiatuczuc.pl/ (mirror: https://rafalsladek.github.io/KwiatUczuc/)
**Repo:** https://github.com/RafalSladek/KwiatUczuc (renamed from `kONtakt`)
**Deployed via:** GitHub Pages from `main` branch, root `/`
**Custom domain:** `kwiatuczuc.pl` via Cloudflare DNS (proxied CNAME → `rafalsladek.github.io`) → GitHub Pages

## Architecture

Single-file app — all HTML, CSS, and JS live in `index.html`:

| File | Purpose |
|---|---|
| `index.html` | Entire app — markup, styles, and logic |
| `CNAME` | Custom domain for GitHub Pages |
| `icons/` | Favicon + app icons (16, 32, 180, 192, 512px + SVG flower of emotions) |
| `scripts/screenshots.js` | Automated screenshots (iPhone SE/14, Pixel 5, Desktop) |
| `scripts/gif.js` | User journey GIF generator (requires ffmpeg) |
| `docs/screenshots/` | Generated screenshots and user-journey.gif |
| `docs/research/` | Reference materials (Feelings Wheel image) |

## Screens

- **Screen 1** — Emotion wheel: 7 core emotions from the Feelings Wheel (Dr. Gloria Willcox) arranged in a circle, with toggle to flower layout. Multi-select emotions (`selectEmo()`), confirm with "confirmBtn" to save today's entry. Rotating prompt text above the wheel (`cycleQuestion()`) cycles through 5 phrasings every 6s. Users can add custom emotions.
- **Screen 2** — Pie chart (`renderPie()`): period selector — dziś/wczoraj/tydzień/2 tyg/4 tyg (`setPeriod()`) — aggregates entries in the selected window into a pie chart with emotion colors.

## Data Storage

All user data is stored **client-side only**:

- `kwiatuczuc_entries` — JSON array of `{date, emotions[]}` entries (one entry per day; `confirmSelection()` overwrites today's entry if it already exists)
- `kwiatuczuc_custom` — JSON array of custom emotion names
- `kwiatuczuc_theme` — `"pastel"` (default) or `"dark"`
- `kwiatuczuc_last_vote` — date string of last vote (shows pie chart on revisit)
- `kwiatuczuc_layout` — `"circle"` or `"flower"` (default)
- `kwiatuczuc_reminder` — reminder time (HH:MM), `"skipped"`, or `"denied"`
- No backend, no sync, no accounts
- Migration from old `kontakt_` prefix runs automatically on load

## Key Implementation Details

- **7 core emotions**: radosc, smutek, wstret, zlosc, strach, dyskomfort, zaskoczenie
- **Colors**: based on original Feelings Wheel — Happy=#F0D860, Sad=#6B9DC8, Disgusted=#7AB48C, Angry=#E88878, Fearful=#A8C878, Bad=#8BA0C0, Surprised=#C8A0D0
- **Two palettes**: dark (rgba 0.28 opacity bg, light text) and pastel (rgba 0.45 opacity bg, dark text). Default is pastel.
- **Two layouts**: circle (equal circles on a ring) and flower (identical oval petals rotated toward center, horizontal counter-rotated text)
- **Circle layout geometry**: `R = (D + gap) / (2 * sin(PI/N))` ensures no overlap
- **Flower layout**: all petals have identical dimensions (arcW x petalLen), each rotated by its angle + 90deg, text counter-rotated to stay horizontal
- **Layout animation**: CSS transitions (0.9s ease) on position, size, and transform — DOM elements persist, only styles change
- **Adding custom emotion**: element appended at center with zero size, then `applyPositions()` triggers animated reflow
- **Emotions shuffled**: `renderWheel()` shuffles baseEmotions on each render
- **Date**: `today()` returns `YYYY-MM-DD` via `toISOString().slice(0,10)`

## PWA

- `manifest.json` + `sw.js` — service worker registered at `index.html:1270` (`/sw.js`)
- Cache strategy: network-first for navigation (HTML), cache-first for other assets; bump `CACHE` version string in `sw.js` when shipping changes that must bust old caches

## Development Notes

- No build tooling — edit `index.html` directly, changes are immediately deployable
- Deploy by pushing to `main` (GitHub Pages auto-builds)
- Do not use `git push --force` on main
- Google Fonts loaded non-blocking (preconnect + preload)
- No `user-scalable=no` in viewport
- `package.json`/`package-lock.json`/`node_modules/` are gitignored — not part of the repo. To run the scripts below, `npm init -y && npm install playwright` locally first (one-time, untracked)

## Scripts

- `node scripts/screenshots.js` — take screenshots for all devices (seeds localStorage with sample entries)
- `node scripts/screenshots.js --device iphonese` — single device
- `node scripts/gif.js` — generate user-journey.gif (requires ffmpeg)
- `node scripts/gif.js --device desktop --fps 4` — custom options

## Known Infrastructure Issue — GH Pages cert vs Cloudflare proxy

**Symptom:** site returns Cloudflare **526 (Invalid SSL Certificate)**.

**Root cause:** `kwiatuczuc.pl` DNS record is a Cloudflare-proxied (orange-cloud) CNAME to `rafalsladek.github.io`. Because it's proxied, GitHub Pages' cert-renewal watchdog sees Cloudflare's rotating anycast IPs on lookup and repeatedly flags the domain as `dns_changed`, resetting its own Let's Encrypt renewal before it completes. Confirm current state: `gh api repos/RafalSladek/KwiatUczuc/pages` → check `https_certificate.state`/`expires_at`. This is structural, not a one-time misconfig — it recurs.

**Current mitigation (temporary):** Cloudflare zone `dc8bc18da9480ee2133c5cd73a23017b` SSL/TLS mode set to **Full** (not Full-strict) — encrypts edge↔origin hop but skips origin cert validation, so requests succeed even while GH's cert is expired/reissuing. Visitor↔edge leg unaffected (valid Cloudflare cert). Acceptable risk for this project — static, no backend, no secrets — but weaker than Full-strict.

**Permanent fix — not yet done, pick one:**
1. **(Recommended)** Grey-cloud the DNS record (Cloudflare dash → DNS → toggle proxy off on the `kwiatuczuc.pl` CNAME) so GH Pages resolves directly and its own Let's Encrypt renews on schedule without interference. Loses Cloudflare CDN/WAF/DDoS layer — acceptable for a static, no-backend site.
2. Migrate hosting from GitHub Pages to Cloudflare Pages — keeps the proxy/CDN, cert lifecycle becomes fully Cloudflare-managed so Full-strict works permanently. Bigger change: updates deploy flow and this doc's "Deployed via" section.
3. Do nothing further — stay on Full mode indefinitely, accept the reduced edge↔origin trust.

Cloudflare API token available in this environment can read zone/DNS state but lacks `zone_settings:edit` — SSL mode and proxy toggle changes require the dashboard.

## Pre-Commit Checklist

1. **Update screenshots**: if any UI change, run `node scripts/screenshots.js` and `node scripts/gif.js`
2. **Visual review**: read screenshots with Read tool, verify layout correctness
3. **Update docs**: if architecture or features changed, update `CLAUDE.md` and `README.md`
