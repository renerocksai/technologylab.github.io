# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the website for **technologylab.ai** — the site for *AI Research & Technology Lab GmbH*, a lab that builds **AI agents**, **AI-powered experiment & lab platforms**, and does **expert engineering**. Built with [Zine](https://zine-ssg.io), a Zig-based static site generator.

The lab ships two products, highlighted on the site and linked out:
- **qualcode.ai** — AI survey coding for open-ended responses (dual independent AI raters, Cohen's kappa / Krippendorff's alpha, reconciliation, exports).
- **labrowser.app** — a research browser that instruments real web behavior as structured data.

There is also a related resource, **readyforagents.ai** (René's thesis + readiness offering: "organisational readiness is most of the work, and we handle it"). It is linked from the home page (a line under the three pillars), the Approach page (the "Readiness is most of the work" section), the Contact page, and the footer.

## Build Commands

```bash
# Local development — serves the site with hot reload (rebuilds on file save).
# Default port 1990; override with --port. Auto-detects non-TTY (no pager).
zine

# Production build → public/ (this is exactly what CI runs).
zine release
```

**Gotcha:** `zine release` refuses to write into a non-empty `public/` ("output directory is not empty"). Either `rm -rf public` first, or pass `-f`/`--force` (force does **not** remove stale files, so prefer `rm -rf public && zine release` for a clean build). `public/` and `zig-out/` are gitignored.

On success `zine release` is quiet and exits 0. Errors print a clear `[script_eval_*]` block naming the layout, line, and column.

## Architecture

**Zine config:** `zine.ziggy` defines site metadata, directory paths, and `static_assets` (files copied verbatim — e.g. `CNAME`, favicons, the long font list). Assets referenced via `$site.asset('name')` are processed automatically and do **not** need a `static_assets` entry.

**Directory structure:**
- `content/` — page content in SuperMD (`.smd`). Frontmatter is Ziggy syntax (`.title`, `.date = @date(...)`, `.author`, `.layout`, `.draft`, `.custom = { ... }`).
- `layouts/` — page templates (`.shtml`, SuperHTML).
- `layouts/templates/` — base template(s) that layouts extend.
- `assets/` — CSS, fonts, images. The only CSS files in use are `theme.css` (the whole design system), `highlight.css` + `term-highlight.css` (code highlighting, included by `post.shtml`).

### Pages & layouts

| URL          | content file        | layout              |
|--------------|---------------------|---------------------|
| `/`          | `content/index.smd` | `home.shtml`        |
| `/approach/` | `content/approach.smd` | `page.shtml`     |
| `/contact/`  | `content/contact.smd` | `contact.shtml`   |
| `/agb/`      | `content/agb.smd`   | `page.shtml`        |
| `/news/`     | `content/news/index.smd` | `collection.shtml` (archive of `subpages()`; empty for now) |
| `/busycard/` | `content/busycard.smd` | `busycard.shtml` |

The home page is **data-driven**: nearly all copy lives in `content/index.smd`'s `.custom` block (`hero_*`, `feature_*`, `project_qualcode_*`, `project_labrowser_*`, `updates_*`), so copy edits rarely touch templates.

`busycard.shtml` is a **fully standalone** page (its own `<!DOCTYPE>`, Tailwind CDN, own theme toggle) — it does **not** extend `base.shtml`. Leave it alone unless working on the personal business card.

### SuperHTML templating — important semantics (learned the hard way)

**`extend` / `<super>`:** `base.shtml` is the skeleton. It defines blocks via elements with an `id` (`title`, `head`, `content`, `footer`) and places a `<super>` inside each block. A child layout (`<extend template="base.shtml">`) provides elements with the **same `id`**; the child's *inner content* is injected at the parent's `<super>`, while the parent's surrounding markup is kept. Every child must declare all four blocks (`title`, `head`, `content`, `footer`) — declare empty ones (e.g. `<footer id="footer"></footer>`) when there's nothing to add.

**`:if` gates CHILDREN, not the tag.** When an `:if` condition is false, SuperHTML still renders the host element's **opening/closing tag** and only drops its children. So putting `:if` directly on a *styled* element (e.g. `<ul class="updates-list" :if=...>`) leaves a stray empty bordered box when false. **Fix:** wrap styled content in a neutral `<div :if="...">…</div>` so the false case renders only an invisible empty `<div></div>`. (`:if` is also the optional-unwrap mechanism — `$if` holds the unwrapped value inside the element.)

**Array/list vs string method names differ:**
- List values like `$page.subpages()` expose **fields** `.len` and `.empty` (no parentheses). Use `$page.subpages().empty.not()` for "has items" and `$page.subpages().empty` for "is empty".
- Strings expose `.len()` as a **method** (with parens), e.g. `$page.description.len().gt(0)`.
- Calling `.len()` on a list errors with `builtin not found`.

**Common directives/expressions in use:** `:text`, `:html`, `:loop="$page.subpages()"` (repeats the element; current item is `$loop.it`, also `$loop.idx/first/last/len`), `$page.custom.get('key')`, optional `$page.custom.get?('key')` and `$page.custom.has('key')`, `$site.page('name')` (resolves a content page by path; `.link()` for its URL, `.subpages()` for collection children), `$site.asset('file').link()`, `$page.date.format('January 02, 2006')`.

## Design system (`assets/theme.css`)

Flat, **editorial** style modeled on the qualcode.ai landing page — slate neutrals + a single azure/indigo blue accent. Rules: **borders, not shadows; no gradients; no glow overlays.** CSS custom properties with a `[data-theme]` light/dark switch; **default is light** (`<html data-theme="light">`, and the inline init script in `base.shtml` picks saved pref → OS dark → light).

- Accent: `--blue-600` (light) / `--blue-400` (dark). Neutrals: Tailwind slate scale.
- Layout is **full-bleed bands**: `.band` / `.band-alt` (alternating bg, 1px `border-block`) and `.band-dark` (inverted closing CTA), each wrapping an inner `.container` (max-width 1100px). There is **no** page-width wrapper.
- Building blocks: `.eyebrow`, `.section-head`/`.section-title`/`.section-subtitle`, `.section-note` (muted section-level link line), three-tier buttons `.btn-primary`/`.btn-secondary`/`.btn-ghost` (primary inverts to white on `.band-dark`), `.feature-grid`/`.feature-card`, `.project-row`(`--reverse`), `.updates-list`/`.empty-state`, `.prose` (≤70ch reading column for content pages), multi-column `.site-footer`.
- Typography: system font stack; headings `font-weight:700; letter-spacing:-0.02em`.

### Content & copy conventions
- Voice is company-first **"we"**; the site is intentionally not personalised (no founder name / personal-blog link in the footer — that was a placeholder and was removed). René's persona lives on readyforagents.ai / the busycard.
- Positioning is **AI agents** (not "AI decision systems") — agents that *act and produce artefacts*, not just advise.
- Avoid copy that presumes the reader's (possibly wrong) beliefs or anxieties (e.g. "you think the model is the problem", "wondering if you're ready"). State value directly.

## Version control — Jujutsu (jj), NOT plain git

**This repo is managed with [Jujutsu](https://github.com/jj-vcs/jj)** (colocated with git). `git status` shows a detached HEAD — that is normal and expected; do not try to "fix" it. **Never create git branches** (`git branch`, `git checkout -b`) here.

Key facts:
- jj **auto-snapshots the entire working copy** into the current change `@` on every command. There is **no staging** — never specify individual files; describing `@` captures all changes.
- The default branch / bookmark is `master`, tracking `origin` (`git@github.com:renerocksai/technologylab.github.io`).

**Standard commit-and-push flow (to `master`):**
```bash
jj describe -m "message"      # describe the current change @ (ALWAYS pass -m; without it, vim opens)
jj bookmark set master -r @   # fast-forward the master bookmark onto @
jj git push                   # pushes the changed tracked bookmark (master)
```
After a successful push the pushed commit becomes immutable, so jj automatically creates a fresh **empty** `@` on top — expected, not an error.

**Gotcha:** `jj commit -m` (and any inline `-m`) chokes on multi-line messages whose lines start with `- ` — clap reads them as flags (`error: unexpected argument '- '`). For bullet-style messages, write the message to a temp file and pass it via command substitution:
```bash
jj describe -m "$(cat /tmp/msg.txt)"
```
End commit messages with the trailer:
`Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`

## Deployment

Pushing `master` triggers `.github/workflows/gh-pages.yml` (`on: push: branches: [master]`, also `workflow_dispatch`), which builds with **Zine v0.10.1** and deploys to GitHub Pages at **https://technologylab.ai** (`CNAME`). Runs typically finish in ~20–30s. Check with `gh run list --workflow=gh-pages.yml` / `gh run watch <id> --exit-status`. (`gh-pages.yml.deactivated` is an inactive variant — ignore it.)

## Verifying visual changes locally

`zine` (dev server) + headless Chrome screenshots is the established way to verify look-and-feel:
```bash
zine --port 1991 &   # then curl http://localhost:1991/ to confirm it's up
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu \
  --no-sandbox --hide-scrollbars --window-size=1280,2700 \
  --screenshot=/tmp/out.png http://localhost:1991/
```
Use **old** `--headless` (not `--headless=new`, which can hang on repeated invocations) and wrap calls in `timeout 30`. To preview dark mode, temporarily flip the init-script fallback in `base.shtml` to `'dark'`, screenshot, then revert (headless Chrome has no saved theme and won't honor a forced `prefers-color-scheme`).
