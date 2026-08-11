# Deploying weifanjiang.github.io

Last verified: August 2026.

## Overview

- **Source** lives on the `master` branch, checked out at `~/weifanjiang.github.io` on zu
  (`ssh weifan@zu.int.seas.harvard.edu`).
- **The live site** is the `gh-pages` branch, served by GitHub Pages at
  <https://weifanjiang.github.io>. Never edit `gh-pages` by hand — it is overwritten on
  every deploy.
- **Deploys are manual**: build the site with Jekyll in a Docker container (a laptop with
  Docker Desktop works; zu's Ruby is too old to build), then push the built `_site` to
  `gh-pages` from zu. The GitHub Actions auto-deploy is currently broken — see
  [Known issues](#known-issues).

## Revising the site

Edit source files on zu's `master` branch:

| What | Where |
|------|-------|
| Bio / homepage | `_pages/about.md` |
| Publications | `_bibliography/papers.bib` |
| CV PDF | `assets/pdf/WeifanJiang_CV.pdf` |
| News items | `_news/` |
| Site-wide settings | `_config.yml` |
| Styles | `_sass/` |

Then commit and push:

```bash
git add -A && git commit -m "..." && git push origin master
```

Pushing `master` does **not** deploy anything (CI is broken). Follow the steps below.

## Deploying

### 1. Build (on a machine with Docker)

```bash
git clone git@github.com:weifanjiang/weifanjiang.github.io.git site && cd site
# or: git pull, in an existing clone

docker run --rm -v "$PWD:/site" -w /site ruby:3.0.2 bash -c "
  apt-get update -qq && apt-get install -y -qq imagemagick
  gem install bundler -v 2.4.22
  bundle install
  JEKYLL_ENV=production bundle exec jekyll build"

tar czf site_build.tgz -C _site .
```

Notes:
- Ruby is pinned at 3.0.2 and gem versions are pinned by the committed `Gemfile.lock`.
- `JEKYLL_ENV=production` matters — without it analytics/minification differ.
- On macOS, start Docker Desktop first (`open -a Docker`; takes a couple of minutes).

### 2. Publish (from zu, which has the GitHub SSH key)

```bash
scp site_build.tgz weifan@zu.int.seas.harvard.edu:/tmp/

ssh weifan@zu.int.seas.harvard.edu
export GIT_SSH_COMMAND="ssh -o UserKnownHostsFile=~/.github_known_hosts"
rm -rf /tmp/ghp
git clone --depth 1 -b gh-pages git@github.com:weifanjiang/weifanjiang.github.io.git /tmp/ghp
cd /tmp/ghp
find . -maxdepth 1 ! -name . ! -name .git -exec rm -rf {} +
tar xzf /tmp/site_build.tgz -C .
touch .nojekyll
git add -fA
git commit -m "deploy: <describe change> [manual deploy]"
git push origin gh-pages
cd / && rm -rf /tmp/ghp /tmp/site_build.tgz
```

### 3. Verify

The change appears at <https://weifanjiang.github.io> within ~1–5 minutes (GitHub Pages
CDN cache is 10 minutes; hard-refresh with Cmd+Shift+R to bypass browser cache).

## Environment quirks on zu

- `~/.ssh` is owned by root and read-only, so GitHub's host keys live in
  `~/.github_known_hosts` instead. The source repo has `core.sshCommand` configured to
  use it; fresh clones (like the `/tmp/ghp` one above) need `GIT_SSH_COMMAND` set
  explicitly, as shown.
- zu's SSH key (`~/.ssh/id_rsa.pub`, "weifan@zu") must be registered at
  <https://github.com/settings/keys>. It was re-added in August 2026 after having been
  removed.

## Known issues

- **GitHub Actions deploy is broken** (as of August 2026): the `bundle exec jekyll build`
  step fails on the runner even though the identical build succeeds in a local
  `ruby:3.0.2` container. The workflow (`.github/workflows/deploy.yml`) was already
  modernized and `Gemfile.lock` pinned; debugging further requires reading the CI logs,
  which needs repo authentication (`gh auth login`). Until fixed, deploys are manual.
- **Do not re-add `polyfill.io`** (was in `_includes/scripts/mathjax.html`): the domain
  was hijacked in 2024 and served malicious sign-in prompts to visitors. Removed in
  August 2026.
