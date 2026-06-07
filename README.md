# liucxer.github.io

Personal developer homepage + per-app marketing / support / privacy pages for all my indie macOS apps, hosted via GitHub Pages.

## Layout

```
liucxer.github.io/                 → Portfolio homepage (this README's repo root)
├── index.html                     → /
├── 4680-weeks/                    → /4680-weeks/
│   ├── index.html                 → marketing
│   ├── support.html               → FAQ / support
│   ├── privacy.html               → privacy policy
│   └── style.css                  → app-specific styles
├── <future-app-slug>/             → next app gets its own subdirectory
│   └── ...
└── README.md
```

Each app is a **self-contained subdirectory** — its own pages, its own `style.css`, its own brand if needed. The root `index.html` is the developer portfolio that links to each app.

## Apps

| App | Live URL | Status |
|---|---|---|
| **4,680 Weeks** | https://liucxer.github.io/4680-weeks/ | macOS · In App Store review |

## App Store Connect URL fields → repo paths

For each app, fill these into App Store Connect:

| Field | URL pattern |
|---|---|
| Marketing URL | `https://liucxer.github.io/<app-slug>/` |
| Support URL | `https://liucxer.github.io/<app-slug>/support.html` |
| Privacy Policy URL | `https://liucxer.github.io/<app-slug>/privacy.html` |

## Adding a new app

1. Create `<app-slug>/` directory at the repo root
2. Drop in `index.html`, `support.html`, `privacy.html`, `style.css` (or copy from an existing app and adapt)
3. Add a new `<article class="app">` block to the root `index.html` linking to it
4. Commit, push, GitHub Pages re-deploys in ~1 minute

## Contact

[liucxer@gmail.com](mailto:liucxer@gmail.com)

---

Site content © 2026 Changxi Liu.
