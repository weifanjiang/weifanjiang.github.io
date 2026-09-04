# Deploying weifanjiang.github.io

Last verified: September 2026.

## Overview

- **Source** lives on the `master` branch. Any clone works; there is also an old checkout
  at `~/weifanjiang.github.io` on zu, which only resolves on Harvard VPN and can drift.
- **The live site** is the `gh-pages` branch, served by GitHub Pages at
  <https://weifanjiang.github.io>. Never edit `gh-pages` by hand — it is overwritten on
  every deploy.
- **Deploys are automatic.** Push to `master` and GitHub Actions
  (`.github/workflows/deploy.yml`) builds the site and publishes it to `gh-pages`.
  Nothing else is required — no Docker, no local Ruby.

## Revising the site

Edit source files on the `master` branch:

| What | Where |
|------|-------|
| Bio / homepage | `_pages/about.md` |
| Publications | `_bibliography/papers.bib` |
| CV PDF | `assets/pdf/WeifanJiang_CV.pdf` |
| News items | `_news/` |
| Site-wide settings | `_config.yml` |
| Styles | `_sass/` |

Then:

```bash
git add -A && git commit -m "..." && git push origin master
```

The site updates within a few minutes. Watch the run in the repo's **Actions** tab; the
change is live once both `deploy` and `pages build and deployment` are green. GitHub's
CDN caches for 10 minutes, so hard-refresh (Cmd+Shift+R) if you still see the old page.

### Adding a publication

1. Add the entry to `_bibliography/papers.bib`. Include `selected = {true}` to show it on
   the homepage, and a `venue = {...}` field with the **full conference name** (e.g.
   `International Conference on Learning Representations`) — that is what renders, not the
   `booktitle`.
2. Put the PDF in `assets/pdf/papers/` and reference it as `pdf = {papers/<file>.pdf}`.
3. **Add the year to `years:` in `_pages/publications.md`.** That list is explicit; a year
   missing from it silently omits the paper from the publications page.

## Known issues and gotchas

- **Do not add `bundler-cache: true` to the deploy workflow.** It installs gems into
  `vendor/bundle`, where Ruby 3.0's built-in `uri` default gem wins activation over the
  version `Gemfile.lock` pins, and the build dies with
  `Gem::LoadError: You have already activated uri 0.10.1, but your Gemfile requires uri 1.1.1`.
  Installing into the default gem path (what the workflow does now) avoids this. Upgrading
  Ruby past 3.0.2 would likely also fix it, but that risks the older pinned gems.
- **The workflow installs ImageMagick explicitly.** `jekyll-imagemagick` needs the
  `convert` binary for responsive images, and the ubuntu-24.04 runner image no longer
  ships it preinstalled. This is why the build broke sometime between 2023 and 2026.
- **Build failures are hard to read without repo auth.** The workflow echoes the head of
  the Jekyll log as a `::error::` annotation, which is readable from the public API
  (`/commits/<sha>/check-runs` → `/check-runs/<id>/annotations`) without a token.
- **Do not re-add `polyfill.io`** (was in `_includes/scripts/mathjax.html`): the domain
  was hijacked in 2024 and served malicious sign-in prompts to visitors. Removed in
  August 2026.

## Fallback: building and publishing by hand

Only needed if Actions is down or you want to preview locally. Requires Docker.

```bash
docker run --rm -v "$PWD:/site" -w /site ruby:3.0.2 bash -c "
  apt-get update -qq && apt-get install -y -qq imagemagick
  gem install bundler -v 2.4.22
  bundle install
  JEKYLL_ENV=production bundle exec jekyll build"
```

To publish that build: clone `gh-pages` to a temp dir, delete everything except `.git`,
copy in the contents of `_site`, `touch .nojekyll`, then commit and push. Publishing from
zu additionally needs
`export GIT_SSH_COMMAND="ssh -o UserKnownHostsFile=~/.github_known_hosts"`, because zu's
`~/.ssh` is root-owned and read-only so GitHub's host keys live outside it.
