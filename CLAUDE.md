# Virtual Crib — CLAUDE.md

## What This Is

Victor Quan's personal portfolio website. Two static HTML pages (`portfolio.html`, `about.html`) served as-is, backed by a Spring Boot service (`spotify-backend/`) that proxies the Spotify API.

## Stack

**Frontend**
- HTML5 / CSS3 / Vanilla JS — no framework, no build step
- Google Fonts CDN — DM Mono, DM Sans, Inter
- Cloudflare Email Protection — email obfuscation script
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
├── portfolio.html            # Landing page — hero, skills, projects, listening widget
├── about.html                # About page — bio, meta info
├── styles/
│   └── main.css              # All styles; shared by both pages
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
- `BASE` — global defaults
- `ANIMATIONS` — keyframes
- `NAV` — navigation bar
- `HERO / ABOUT / SKILLS / PROJECTS / FOOTER` — Skills and Projects sections exist in CSS but are removed from `portfolio.html`
- `NEWS TICKER` — fixed scrolling bar at top (font size 0.88rem)
- `LISTENING / SPOTIFY` — now-playing + recently-played widget (card layout: album art stacked above text, centered; card max-width 240px; art 180×180px)
- `SCROLL REVEAL` — IntersectionObserver fade-in
- `CURSOR` — custom dot cursor
- `RESPONSIVE` — single breakpoint at `768px`

### Design tokens (`:root`)

| Variable | Value | Use |
|---|---|---|
| `--bg` | `#F7F6F2` | Page background |
| `--fg` | `#111110` | Primary text |
| `--muted` | `#888884` | Secondary text |
| `--accent` | `#1A1A18` | Dark accent |
| `--line` | `#E2E0D8` | Borders |
| `--tag-bg` | `#EEEEE9` | Skill/tech tags |
| `--ticker-gold` | `#D4A017` | News ticker highlight |

## JavaScript Patterns

All JS is inline in the HTML files:
- **Custom cursor** — mousemove listener on `.cursor`
- **Scroll reveal** — `IntersectionObserver` adds `.visible` class
- **Spotify widget** — fetches `/spotify/now-playing` and `/spotify/recently-played` on load, polls now-playing every 30s. Backend URL is `const SPOTIFY_API` at the top of the script block in `portfolio.html`. Currently set to the Railway URL. `progressMs` is extracted at the top level of the now-playing response and passed through to seed the elapsed timer accurately.
  - **Paused track logic:** When now-playing returns `isPlaying: false` but still has track data (paused song), the frontend shows that track as "Recently Played" directly instead of fetching the `/recently-played` endpoint (which is cached 120s and may return a stale/different song).
  - **Dynamic label:** The `#spotify-label` element above the widget switches between "Currently Listening" and "Recently Played" based on playback state.
  - **Admin controls:** Hidden by default; activated via Shift+S pin input. Shows prev/play-pause/next buttons.

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
- Spotify widget live in `portfolio.html` — fetches now-playing + recently-played from Railway backend, polls every 30s, elapsed timer seeded from `progressMs`
- News ticker endpoint (`/news/headlines`) implemented; `NEWS_API_KEY` set locally via `application-local.properties`
- Footer redesigned: centered, SVG icons (GitHub, LinkedIn, Twitter/X, Email) + `Victor Quan · © 2026 · Portfolio` text line
- Hero: italic DM Sans tagline, frosted glass Spotify card (warm beige tint)

**Layout design:**
- Single-viewport layout: `body` has `height: 100vh` + `overflow: hidden` — no scrolling, all content visible at once
- `body` padding-top: 90px (clears fixed ticker + fixed nav)
- `#hero` uses `flex: 1` to fill remaining vertical space; content is vertically centered
- Spotify card: album art (180×180) stacked above centered text, max-width 240px
- Playback controls: 24×24 SVG icons, 1.5rem gap between buttons
- Footer padding: 1.5rem

**Still needs personalization:**
- `about.html` — bio text, location, years of experience, real company names
- `portfolio.html` — real GitHub/LinkedIn/Twitter URLs in footer icons

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
