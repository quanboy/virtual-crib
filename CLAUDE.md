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
- Caffeine caching (30s now-playing, 120s recently-played)
- Deployed on Railway

## Development

**Frontend** — `serve` is installed but not on PATH, use:
```bash
npx serve -p 3000 .
# or the full path:
C:\Users\shuiw\AppData\Roaming\npm\node.exe C:\Users\shuiw\AppData\Roaming\npm\node_modules\serve\bin\serve.js -p 3000 .
```

**Backend:**
```bash
cd spotify-backend
mvn spring-boot:run
```
Requires `SPOTIFY_CLIENT_ID` and `SPOTIFY_CLIENT_SECRET` env vars to start. `SPOTIFY_REFRESH_TOKEN` is optional on first boot — get it by running the OAuth flow (see below).

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
│       │   │   └── SpotifyController.java  # API endpoints (/spotify/*)
│       │   ├── service/
│       │   │   ├── SpotifyService.java     # Spotify API calls + caching
│       │   │   └── SpotifyTokenService.java # Token refresh (scheduled + on-demand)
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
- `HERO / ABOUT / SKILLS / PROJECTS / FOOTER`
- `NEWS TICKER` — fixed scrolling bar at top
- `LISTENING / SPOTIFY` — now-playing + recently-played widget
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
- **Spotify widget** — fetches `/spotify/now-playing` and `/spotify/recently-played` on load, polls now-playing every 30s. Backend URL is `const SPOTIFY_API` at the top of the script block in `portfolio.html`. Currently set to the Railway URL.

## Spotify API Endpoints

| Endpoint | Returns | Cache |
|---|---|---|
| `GET /spotify/now-playing` | Current track or `{isPlaying: false}` | 30s |
| `GET /spotify/recently-played` | Last 6 tracks with `playedAt` | 120s |
| `GET /spotify/random-track` | Random track from saved library | None |

Track object fields: `isPlaying`, `title`, `id`, `artist`, `album`, `albumArt`, `url`, `previewUrl`, `durationMs`.

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
| `PORT` | No | `8080` | Railway sets this automatically |

## Status

**Working:** Spotify widget live in `portfolio.html` — fetches now-playing + recently-played from Railway backend. Footer copyright updated to Victor Quan 2026.

**Still needs personalization:**
- `about.html` — bio text (`[your current project or role]`, `[Company A]`, `[Company B]`, interests), location, years of experience
- `portfolio.html` — project names, descriptions, links (currently Alpha/Beta/Gamma/Delta), GitHub/LinkedIn/Twitter URLs in footer

**Still needs doing:**
- Deploy frontend to static host, then set `ALLOWED_ORIGINS` in Railway
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
