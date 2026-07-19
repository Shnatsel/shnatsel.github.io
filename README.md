# Sergey "Shnatsel" Davidoff

Personal blog built with [Zola](https://www.getzola.org/) and the [Pickles theme](https://github.com/lukehsiao/zola-pickles), deployed to GitHub Pages.

## Local Preview

Install Zola 0.22.1 or newer, then run:

```bash
zola serve --interface 127.0.0.1 --port 1111
```

Open <http://127.0.0.1:1111/>.

## Publishing

The GitHub Actions workflow builds the site from Markdown on every push to `main` and deploys the generated `public/` output to GitHub Pages.

One-time GitHub setup:

1. Create an empty GitHub repository named `shnatsel.github.io`.
2. In repository settings, set Pages source to **GitHub Actions**.
3. Add the remote and push:

```bash
git remote add origin git@github.com:Shnatsel/shnatsel.github.io.git
git push -u origin main
```

## Adding Posts

Add a Markdown file directly under `content/` with TOML front matter:

```markdown
+++
title = "Post title"
date = 2026-07-19
description = "One-sentence summary."

[taxonomies]
tags = ["Rust"]

[extra]
author = 'Sergey "Shnatsel" Davidoff'
+++

Post body here.
```

Then commit and push to `main`.

## Theme Updates

Pickles is included as a Git submodule:

```bash
git submodule update --init --recursive
git submodule update --remote themes/zola-pickles
```
