# AGENTS.md — Blog Generation & Publishing Guide

This repository is a [Hexo](https://hexo.io/) static blog using the
[hexo-theme-flatpaper](https://github.com/Homulilly/hexo-theme-flatpaper) theme.
It is deployed to GitHub Pages at **https://zhootoo.github.io/**.

## Repository layout

- `main` branch — blog source (this working tree). Safe to commit here.
- `gh-pages` branch — generated static site, produced by `hexo deploy`.
  Do NOT edit `gh-pages` manually.
- `source/_posts/` — Markdown articles. This is where new posts go.
- `source/` — pages (e.g. `about/index.md`) and assets (`images/`).
- `_config.yml` — site + deployment config (url, root, deploy target).
- `_config.flatpaper.yml` — theme override config.
- `themes/flatpaper/` — the theme (committed directly, no nested .git).

## Branch & remote

- Remote `origin` = `https://github.com/zhootoo/zhootoo.github.io.git`
- Always work on `main`. Deploy uses `gh-pages` automatically.

## How to create a new post

```bash
npx hexo new post "Post Title"
```

This creates `source/_posts/Post Title.md`. Edit the front-matter and body:

```markdown
---
title: Post Title
date: 2026-08-15 12:00:00
categories: [Tech]
tags: [Hexo, Blog]
---

Write content in Markdown here.
```

Front-matter fields `categories` / `tags` feed the menu's category/tag/archive pages.

### Images in posts (post_asset_folder is enabled)

`post_asset_folder: true` is set, so `hexo new post` also creates a folder
`source/_posts/<title>/` next to the Markdown file. Put images there and
reference them with a relative path:

```markdown
![caption](image.png)
```

or with the Hexo tag:

```markdown
{% asset_img image.png caption %}
```

Both keep images bundled with the post and backed up in `main`. Do NOT use
absolute `/images/...` paths for post images unless the file lives in
`source/images/`. External URLs are allowed but not recommended for durability.

## How to publish

```bash
npx hexo deploy
```

`hexo deploy` runs `hexo generate` then pushes the result to `gh-pages`.
Wait ~1 minute, then view **https://zhootoo.github.io/**.

Optional local preview before publishing:

```bash
npx hexo server   # http://localhost:4000  (Ctrl+C to stop)
```

Do NOT run `hexo clean` routinely — on Windows it can fail deleting
`.deploy_git`. Only use it when a full rebuild is truly needed.

## How to back up source (recommended after writing)

```bash
git add .
git commit -m "Add new post: <title>"
git push origin main
```

## Commit message rules (IMPORTANT)

- Use **English only**. Windows PowerShell passes Chinese to git as GBK,
  producing mojibake in commit messages.
- Examples: `Add new post: <title>`, `Update config`, `Blog source backup`,
  `Site updated`.
- Keep them short and descriptive.
- Never use non-ASCII characters in commit messages.

## Deployment config (already set in `_config.yml`)

```yaml
url: https://zhootoo.github.io
root: /
deploy:
  type: git
  repo: https://github.com/zhootoo/zhootoo.github.io.git
  branch: gh-pages
  message: "Site updated"
```

## GitHub Pages activation (one-time, manual)

In the repo Settings → Pages, set Source = "Deploy from a branch",
branch = `gh-pages`, folder = `/ (root)`, then Save.

## Notes

- `node_modules/`, `public/`, `db.json` are git-ignored.
- The theme lives under `themes/flatpaper/` and is committed as normal files
  (its original `.git` was removed to avoid a nested repo).
- Social links, avatar, menu, and nav are configured in `_config.flatpaper.yml`.
