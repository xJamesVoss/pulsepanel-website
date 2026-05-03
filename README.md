# PulsePanel marketing site

Static site for [pulsepanelapp.com](https://pulsepanelapp.com), the marketing site for PulsePanel — a Minecraft server admin app for iOS.

## Stack

Plain HTML, CSS, and JavaScript. No build step. Hosted on GitHub Pages.

## Structure

- `index.html` — landing page with hero dashboard, features grid, screen carousel, setup teaser, FAQ
- `setup.html` — full setup guide for installing PulseConnect and connecting the app
- `privacy.html` — privacy policy
- `resources.html` — placeholder for documentation, plugin downloads, changelog
- `terms.html` — placeholder for terms of service
- `Screenshots/` — app screenshots from the iOS simulator
- `CNAME` — tells GitHub Pages the custom domain

## Development

Open `index.html` directly in a browser, or run any local static server in this directory.

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Deployment

Push to `main`. GitHub Pages auto-deploys.
