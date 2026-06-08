# Tech Blog

Personal tech blog — kernel debugging, performance analysis, distributed systems.

**Published site**: https://liucxer.github.io/blog/

每篇文章同时提供 **中文** 和 **English** 版本。
Each post is available in both **中文** and **English**.

## Posts

| Date | Title | 中文 | English | Tags |
|---|---|---|---|---|
| 2026-06-08 | A Deep Dive into a 1-Second Linux bcache Write Stall — root btree Lock Contention | [zh](2026-06-08-bcache-root-lock-spike.zh.html) ([md](2026-06-08-bcache-root-lock-spike.zh.md)) | [en](2026-06-08-bcache-root-lock-spike.en.html) ([md](2026-06-08-bcache-root-lock-spike.en.md)) | `linux-kernel` `bcache` `ceph` `bpf` `rwsem` |

## Conventions

- **Bilingual**: each post is authored in both languages, kept in sync
- **Filename**: `YYYY-MM-DD-slug.<lang>.<ext>` where `<lang>` ∈ {`zh`, `en`}, `<ext>` ∈ {`md`, `html`}
- **Source**: `.md` (editable)
- **Render**: `.html` (served via GitHub Pages, shares `style.css`)
- **Language switcher**: each post has a `中文 · English` toggle at the top
- **Index**: `index.html` lists all posts with both language links
- **Content**: all real production data, desensitized

## Build workflow

1. Write Chinese draft as `YYYY-MM-DD-slug.zh.md`
2. Translate to English as `YYYY-MM-DD-slug.en.md` (or vice versa)
3. Run the build script to generate both `.html` files (with shared chrome + lang switcher)
4. Add entry to `index.html` and this `README.md`
5. Commit + push

---

[← Back to portfolio](../)
