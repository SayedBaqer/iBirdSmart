# iBird Hatching

Offline-first PWA for incubation batch tracking — schedule, guidance,
egg testing, and cost economics. See
[incubation-platform-dev-plan.md](incubation-platform-dev-plan.md) for
the full product plan.

No build step — plain HTML/CSS/JS, served as static files.

## Run locally

```
python -m http.server 8080
```

Open http://localhost:8080/index.html

## Deploy (temporary, via GitHub Pages)

Settings → Pages → Deploy from branch → `main` / `/ (root)`.
(Private repos need GitHub Pro/Team for Pages; otherwise use a host
that supports private-repo deploys, e.g. Cloudflare Pages or Vercel,
until this moves to the production domain.)
