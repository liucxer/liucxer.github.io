# liucxer.github.io

Personal developer homepage + per-app marketing / support / privacy pages for all my indie macOS apps, hosted via GitHub Pages.

## Layout

```
liucxer.github.io/                 → Portfolio homepage (this README's repo root)
├── index.html                     → /
├── 4680-weeks/                    → /4680-weeks/         (macOS · English)
├── claude-desk/                   → /claude-desk/        (macOS · English)
├── learning-flashcards/           → /learning-flashcards/ (iOS · 中文)
├── dingding-logic/                → /dingding-logic/      (iOS · 中文)
├── first-principles/              → /first-principles/   (web · 中文原型)
├── preowned-islands/              → /preowned-islands/    (iOS · 中文)
├── preowned-islands-intl/         → /preowned-islands-intl/ (iOS · English)
│   ├── index.html                 → marketing
│   ├── support.html               → FAQ / support
│   ├── privacy.html               → privacy policy
│   └── style.css                  → app-specific styles
└── README.md
```

Each app is a **self-contained subdirectory** — its own pages, its own `style.css`, its own brand if needed. The root `index.html` is the developer portfolio that links to each app. Page language follows the app's primary market (English for the macOS apps; 中文 for the kids' iOS apps).

## Apps

| App | Live URL | Platform / Language | Status |
|---|---|---|---|
| **4,680 Weeks** | https://liucxer.github.io/4680-weeks/ | macOS · English | In App Store review |
| **claude-desk** | https://liucxer.github.io/claude-desk/ | macOS · English | Early development |
| **小豆丁学堂** | https://liucxer.github.io/learning-flashcards/ | iOS · 中文 | 免费 · [App Store](https://apps.apple.com/cn/app/小豆丁学堂/id6762811519) |
| **小豆丁逻辑课** | https://liucxer.github.io/dingding-logic/ | iOS · 中文 | 免费 · [App Store](https://apps.apple.com/cn/app/小豆丁-逻辑课/id6775833119) |
| **第一性原理** | https://liucxer.github.io/first-principles/ | web · 中文 | 产品原型 |
| **二手海岛** | https://liucxer.github.io/preowned-islands/ | iOS · 中文 | ¥18 · App Store(审核中) |
| **PreOwned Islands** | https://liucxer.github.io/preowned-islands-intl/ | iOS · English | $2.99 · App Store (coming soon) |

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
