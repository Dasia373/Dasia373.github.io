# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Hugo static blog (Blowfish theme) deployed to GitHub Pages at `dasia373.github.io`. Content is written in Chinese (zh-cn) and covers embedded Linux, research notes, and learning logs.

## Commands

```bash
hugo server          # Start dev server at http://localhost:1313 (Drafts visible)
hugo server -D       # Start dev server including draft posts
hugo                 # Build site to ./public/
hugo new posts/xxx.md  # Create a new post from archetype
```

Push to `main` triggers `.github/workflows/deploy.yml`, which builds with Hugo Extended and deploys `./public/` to the `gh-pages` branch via `peaceiris/actions-gh-pages`.

## Architecture

- **Theme**: [blowfish](https://github.com/nunocoracao/blowfish) (submodule at `themes/blowfish`). `themes/PaperMod` is also a submodule but unused.
- **Config**: Split under `config/_default/` — `hugo.toml` (site-level), `params.toml` (theme params), `languages.zh-cn.toml` (language strings). The root `hugo.toml` is largely redundant after the config split.
- **Custom homepage**: `layouts/partials/home/custom.html` — renders a hero section (avatar, title, subtitle, description) followed by the 10 most recent posts as styled cards. Activated by `homepage.layout = "custom"` in params.
- **Custom CSS**: `assets/css/custom.css` — full light/dark mode theme with CSS custom properties. Dark mode defaults on, with auto-switch per browser preference. Overrides many Blowfish defaults via `!important`.
- **Content**:
  - `content/posts/` — all blog posts (markdown with TOML frontmatter: `title`, `date`, `draft`, `tags`)
  - `content/about/` — About page
  - `content/_index.md` — Homepage intro text
- **Static files**: `static/images/avatar.jpg` and `static/images/bg.jpeg`
- **Archetype**: `archetypes/default.md` — template for `hugo new`, posts default to `draft: true`

## Key conventions

- Posts **must** set `draft: false` to appear on the published site
- Commit message style: `add: xxx`, `update: xxx`, `fix: xxx`, `config: xxx`
- The root-level `hugo.toml` has `baseURL = "https://dasia373.github.io/"` — if the domain changes, update it in both `hugo.toml` and `config/_default/hugo.toml`
