# Virtual Crib — CLAUDE.md

## What This Is

Victor Quan's personal portfolio website. Single-page layout in `portfolio.html` with Hero, About, and Projects sections. Backed by a Spring Boot service (`spotify-backend/`) that proxies the Spotify API.

## Stack

**Frontend**
- HTML5 / CSS3 / Vanilla JS — no framework, no build step
- Google Fonts CDN — DM Mono, DM Sans, Inter
- `serve` (Node.js) — local dev server only

**Backend** (`spotify-backend/`)
- Spring Boot 3.2 / Java 17
- Caffeine caching (10s now-playing, 120s recently-played, 600s headlines)
- Deployed on Railway

## Development

**Frontend** — `serve` is installed but not on PATH, use:
```bash
npx serve -p 3000 .
# or the full path:
C:\Users\shuiw\AppData\Roaming\npm\node.exe C:\Users\shuiw\AppData\Roaming\npm\node_modules\serve\bin\serve.js -p 3000 .
```

**Backend:**
```powershell
cd spotify-backend
mvn spring-boot:run "-Dspring-boot.run.profiles=local"
```
Requires `SPOTIFY_CLIENT_ID` and `SPOTIFY_CLIENT_SECRET` env vars to start. `SPOTIFY_REFRESH_TOKEN` is optional on first boot — get it by running the OAuth flow (see below).

`mvn` is not on PATH by default. Either use the full path or add it once (no admin needed):
```powershell
[System.Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\Program Files\JetBrains\IntelliJ IDEA 2025.3.3\plugins\maven\lib\maven3\bin", "User")
```
Then reopen your terminal.

Local secrets go in `spotify-backend/src/main/resources/application-local.properties` (gitignored). That file is loaded automatically when running with `-Dspring-boot.run.profiles=local`.

## File Layout

```
virtual-crib/
├── portfolio.html            # Single-page site — hero, about, projects, listening widget
├── styles/
│   └── main.css              # All styles
├── spotify-backend/          # Separate git repo, deployed to Railway
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/victorquan/spotify/
│       │   ├── controller/
│       │   │   ├── AuthController.java     # OAuth flow (/auth/login, /auth/callback)
│       │   │   ├── SpotifyController.java  # API endpoints (/spotify/*)
│       │   │   └── NewsController.java     # GET /news/headlines
│       │   ├── service/
│       │   │   ├── SpotifyService.java     # Spotify API calls + caching
│       │   │   ├── SpotifyTokenService.java # Token refresh (scheduled + on-demand)
│       │   │   └── NewsService.java        # Fetches top headlines from newsapi.org
│       │   └── config/
│       │       └── SpotifyConfig.java      # Caffeine cache + CORS config
│       ├── main/resources/application.properties
│       └── test/java/com/victorquan/spotify/service/
│           ├── SpotifyServiceTest.java
│           └── SpotifyTokenServiceTest.java
└── .claude/
    ├── launch.json
    └── settings.local.json
```

## CSS Architecture

All styles in `styles/main.css`, organized by section comment headers:

- `RESET & VARIABLES` — CSS custom properties
- `ANIMATIONS` — keyframes (fadeIn, blink, tickerScroll, blinkCaret)
- `NAV LINKS` — `.nav-sub` used in hero-links
- `SECTIONS` — shared section padding and labels
- `HERO` — heading with typewriter effect, nav links, tagline, Spotify widget
- `ABOUT` — bio text + meta grid
- `PROJECTS` — masonry card grid (2-column, `break-inside: avoid`)
- `FOOTER` — social icons + copyright, pinned to bottom via `margin-top: auto`
- `NEWS TICKER` — fixed bar at top with fade-in + expand load animation
- `SCROLL REVEAL` — IntersectionObserver fade-up with staggered delays
- `TYPEWRITER` — blinking caret animation for hero heading
- `CURSOR` — custom dot cursor (hidden on touch devices)
- `LISTENING / SPOTIFY` — dark card widget with gradient background, rounded corners, gold left border, edge-to-edge album art, frosted glass backdrop, soft shadow. Track info, elapsed timer, admin controls
- `RESPONSIVE` — single breakpoint at `768px`

### Design tokens (`:root`)

| Variable | Value | Use |
|---|---|---|
| `--bg` | `#F7F6F2` | Page background |
| `--fg` | `#111110` | Primary text |
| `--muted` | `#888884` | Secondary text |
| `--accent` | `#1A1A18` | Dark accent (hero divider) |
| `--line` | `#E2E0D8` | Borders |
| `--ticker-gold` | `#D4A017` | News ticker highlight |

## JavaScript Patterns

All JS is inline in `portfolio.html`:
- **Custom cursor** — mousemove listener on `.cursor`
- **Typewriter** — heading and tagline type simultaneously at 60ms/char via shared `typeText()` helper. Caret removed on completion.
- **Scroll reveal** — `IntersectionObserver` adds `.visible` class; project cards staggered 150ms apart. Elements in viewport on load are skipped until user scrolls.
- **Spotify widget** — fetches `/spotify/now-playing` and `/spotify/recently-played` on load, polls now-playing every 10s. Backend URL is `const SPOTIFY_API` at the top of the script block. Currently set to the Railway URL. `progressMs` seeds the elapsed timer accurately. First load triggers a slide-up animation (one-time via `_widgetLoaded` flag).
  - **Dynamic label:** `#spotify-label` starts hidden (`visibility: hidden`), shown via `setLabel()` helper once API responds. Switches between "Currently Listening" and "Recently Played".
  - **Admin controls:** Hidden by default; activated via Shift+S pin input (`101901`). Shows prev/play-pause/next buttons. Play/pause button toggles based on `_isCurrentlyPlaying`. Polling pauses while pin input is visible.
- **News ticker** — fetches `/news/headlines`, duplicates items for seamless scroll, ticker bar uses fade-in + expand animation on load via `.ticker-visible` class. Refreshes every 5 minutes.

## Spotify API Endpoints

| Endpoint | Returns | Cache |
|---|---|---|
| `GET /spotify/now-playing` | Current track or `{isPlaying: false}` | 10s |
| `GET /spotify/recently-played` | Last 6 tracks with `playedAt` | 120s |
| `GET /spotify/random-track` | Random track from saved library | None |

Track object fields: `isPlaying`, `title`, `id`, `artist`, `album`, `albumArt`, `url`, `previewUrl`, `durationMs`, `progressMs` (now-playing only).

## News API Endpoints

| Endpoint | Returns | Cache |
|---|---|---|
| `GET /news/headlines` | Up to 15 US top headline strings | 600s |

Powered by [newsapi.org](https://newsapi.org). Returns an empty array if `NEWS_API_KEY` is not set (ticker stays hidden).

## OAuth Flow (one-time setup)

**Already completed.** `SPOTIFY_REFRESH_TOKEN` is set in Railway. Backend is live and serving data.

If you ever need to redo it (e.g. token revoked, new Railway service):
1. Set `SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET`, `SPOTIFY_CALLBACK_URL` in Railway
2. Visit `https://spotify-backend-production-265b.up.railway.app/auth/login` → authorize on Spotify
3. Callback page displays the refresh token — copy it
4. Add it to Railway as `SPOTIFY_REFRESH_TOKEN` → redeploys automatically
5. Done — token auto-refreshes every 55 minutes

The running instance is updated immediately after step 2 (no restart needed); env var in step 4 is for persistence across restarts.

**Known gotchas encountered during setup:**
- `SPOTIFY_REFRESH_TOKEN` has no default — app crashes on start if missing. Fixed: `${SPOTIFY_REFRESH_TOKEN:}` in `application.properties`
- `redirect_uri` in the login URL must be URL-encoded (`URLEncoder.encode`). Fixed in `AuthController.java`
- Spotify dashboard: after typing a redirect URI you must click the inline **Add** button before clicking **Save**, otherwise it doesn't save
- Redirect URIs must use `https://` for non-localhost URLs
- `SPOTIFY_CALLBACK_URL` must be set in Railway — without it the backend defaults to `http://localhost:8080/auth/callback` and Spotify rejects it

## Railway Environment Variables

| Variable | Required | Default | Notes |
|---|---|---|---|
| `SPOTIFY_CLIENT_ID` | Yes | — | From Spotify developer dashboard |
| `SPOTIFY_CLIENT_SECRET` | Yes | — | From Spotify developer dashboard |
| `SPOTIFY_REFRESH_TOKEN` | No* | `""` | *App starts without it; get via OAuth flow |
| `SPOTIFY_CALLBACK_URL` | Yes (prod) | `http://localhost:8080/auth/callback` | Must match Spotify dashboard exactly |
| `ALLOWED_ORIGINS` | Yes (prod) | `http://localhost:3000` | Set to portfolio's deployed domain |
| `NEWS_API_KEY` | No | `""` | Free key from newsapi.org; ticker hidden if missing |
| `PORT` | No | `8080` | Railway sets this automatically |

## Status

**Working:**
- Single-page layout: hero → about → projects → footer, all in `portfolio.html`
- "About" and "Projects" are anchor links below the hero heading with accent bar divider
- Typewriter effect on hero heading
- Spotify widget live — dark card with album art stacked above text, polls every 10s, elapsed timer, slide-up load animation
- Admin controls (Shift+S) with prev/play-pause/next
- News ticker at top with fade-in + expand load animation, scrolls continuously
- About section with bio text and meta info (Location: Orlando, FL; Focus: Full-Stack Development; Currently: Data Coordinator)
- Projects section with 5 projects in masonry card layout with hover glow + lift
- Scroll reveal animations on about grid and project cards (staggered)
- Footer: centered SVG icons (GitHub, LinkedIn, Email) + copyright line
- Social links: GitHub (quanboy), LinkedIn (victor-quan-729752176), Email (victorquan24@gmail.com)

**Still needs doing:**
- Deploy frontend to static host, then set `ALLOWED_ORIGINS` in Railway
- Add `NEWS_API_KEY` to Railway environment variables so ticker works in production
- Run `mvn test` to verify unit tests pass against latest backend code

## Deployment

**Backend** — Railway, connected to the `spotify-backend/` git repo. Push to `main` triggers a redeploy. Has a `Dockerfile` for the build.

**Frontend** — not yet deployed. Static files can go on any static host (GitHub Pages, Netlify, Vercel, Cloudflare Pages). No build step needed. When deployed, set `ALLOWED_ORIGINS` on Railway to the live domain (comma-separated if multiple).

## Tests

Unit tests in `spotify-backend/src/test/` using JUnit 5 + Mockito (no Spring context):
```bash
cd spotify-backend
mvn test
```
