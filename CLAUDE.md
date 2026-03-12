# Virtual Crib — CLAUDE.md

## What This Is

A personal portfolio website — two static HTML pages (`portfolio.html`, `about.html`) styled with a single shared CSS file. No framework, no build step, no backend.

## Stack

- **HTML5 / CSS3 / Vanilla JS** — no dependencies, no transpilation
- **Google Fonts CDN** — DM Mono, DM Sans, Inter
- **Cloudflare Email Protection** — email obfuscation script
- **`serve` (Node.js)** — local dev server only, not part of the site itself

## Development

```bash
serve -p 3000 .
```

Configured in `.claude/launch.json` to use `C:\Users\shuiw\AppData\Roaming\npm\node.exe`.

## File Layout

```
virtual-crib/
├── portfolio.html       # Landing page — hero, skills grid, project cards
├── about.html           # About page — bio, meta info
├── styles/
│   └── main.css         # All styles; shared by both pages
└── .claude/
    ├── launch.json      # Dev server config
    └── settings.local.json
```

## CSS Architecture

All styles live in `styles/main.css`, organized by section with comment headers:

- `RESET & VARIABLES` — CSS custom properties (colors, fonts)
- `BASE` — global defaults
- `ANIMATIONS` — keyframes
- `NAV` — navigation bar
- `HERO / ABOUT / SKILLS / PROJECTS / FOOTER`
- `NEWS TICKER` — fixed scrolling bar at top
- `SCROLL REVEAL` — Intersection Observer fade-in
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

All JS is inline in the HTML files. Current usage:
- **Custom cursor** — mouse move listener updates a `.cursor-dot` element
- **Scroll reveal** — `IntersectionObserver` adds an `is-visible` class to animate elements in
- No module system, no bundler

## Content Placeholders

Both pages still have template text that needs personalizing:

- `about.html` — bio text, company names, interests, location, years of experience
- `portfolio.html` — project names, descriptions, and links (currently Alpha/Beta/Gamma/Delta)
- Both — `© 2025 Your Name. Built with care.` in the footer

## Deployment

No deployment config exists. The site is a pure static build — drop the files on any static host (GitHub Pages, Netlify, Vercel, Cloudflare Pages, S3).

## No Tests, No Build

There is no test suite, no CI/CD pipeline, no bundler, and no linter configured. Changes are reflected immediately in the browser.
