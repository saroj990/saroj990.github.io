# Saroj Blog

Personal blog and portfolio site built with [Hugo](https://gohugo.io/) and deployed to [GitHub Pages](https://pages.github.com/).

**Live site:** [https://saroj990.github.io/](https://saroj990.github.io/)

## About

This repository contains the source for my personal blog. I write about software engineering, backend development, system design, AI tools, and lessons learned from building products.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Static site generator | [Hugo](https://gohugo.io/) |
| Theme | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) |
| Hosting | GitHub Pages |
| CI/CD | GitHub Actions |

## Project Structure

```
.
├── archetypes/          # Templates for new content
├── content/
│   ├── about/           # About page
│   ├── contact/         # Contact page
│   ├── posts/           # Blog posts (Markdown)
│   └── projects/        # Projects page
├── themes/
│   └── PaperMod/        # Hugo theme (git submodule)
├── hugo.toml            # Site configuration
└── .github/workflows/   # Deployment workflow
```

## Local Development

### Prerequisites

- [Hugo](https://gohugo.io/installation/) (extended edition — required for the PaperMod theme)
- [Git](https://git-scm.com/)

### Install Hugo

This project uses the **extended** edition of Hugo (needed for SCSS/SASS support in PaperMod). Install it before running the site locally.

#### macOS (Homebrew)

If you don't have [Homebrew](https://brew.sh/), install it first:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Then install Hugo:

```bash
brew install hugo
```

#### Linux (Homebrew)

Homebrew also works on Linux:

```bash
brew install hugo
```

Or use your distro's package manager if available — see the [official Linux install guide](https://gohugo.io/installation/linux/).

#### Windows

Using [Chocolatey](https://chocolatey.org/):

```powershell
choco install hugo-extended
```

Or using [Scoop](https://scoop.sh/):

```powershell
scoop install hugo-extended
```

See the [official Windows install guide](https://gohugo.io/installation/windows/) for other options.

#### Verify installation

Confirm Hugo is installed and that the output includes `extended`:

```bash
hugo version
```

Example output:

```
hugo v0.147.0+extended darwin/arm64 BuildDate=...
```

If `extended` is missing, reinstall using one of the methods above or download the `hugo_extended` binary from [Hugo releases](https://github.com/gohugoio/hugo/releases).

### Setup

Clone the repository and initialize the theme submodule:

```bash
git clone --recurse-submodules git@github.com:saroj990/saroj990.github.io.git
cd saroj990.github.io
```

If you already cloned without submodules (required — the site will not work without this):

```bash
git submodule update --init --recursive
```

Verify the theme is present:

```bash
ls themes/PaperMod
```

You should see theme files (`layouts/`, `assets/`, etc.), not an empty folder.

### Run Locally

Start the development server with live reload:

```bash
hugo server -D --baseURL http://localhost:1313/
```

Open [http://localhost:1313](http://localhost:1313) in your browser.

The `--baseURL` flag ensures links work correctly during local development (without it, menu links may point to the production site).

### Build for Production

```bash
hugo --minify
```

The generated site is written to the `public/` directory.

## Writing Posts

Create a new post from the archetype:

```bash
hugo new posts/my-new-post.md
```

Edit the generated file in `content/posts/`. Set `draft = false` in the front matter when you're ready to publish.

Example front matter:

```yaml
---
date: '2026-06-24T12:00:00+05:30'
draft: false
title: 'My New Post'
---
```

## Deployment

Pushes to the `master` branch trigger an automatic deployment via GitHub Actions (`.github/workflows/hugo.yml`).

The workflow:

1. Checks out the repository (including submodules)
2. Installs Hugo
3. Builds the site with `hugo --minify`
4. Deploys the `public/` output to GitHub Pages

No manual deployment steps are required — commit and push to `master` to publish.

## Troubleshooting

### "Page not found" on localhost

This usually means the **PaperMod theme submodule** was not downloaded. The `themes/PaperMod` folder must contain the theme files, not be empty.

Fix it:

```bash
git submodule update --init --recursive
```

Then **restart** the Hugo server (stop it with `Ctrl+C`, then run `hugo server -D` again).

### Hugo fails with "module PaperMod not found"

Same fix as above — run the submodule init command and restart the server.

## Configuration

Site settings live in `hugo.toml`, including:

- Site title, base URL, and locale
- Navigation menu (Posts, Projects, About, Contact)
- Theme options (dark mode, reading time, code copy buttons)
- Social links (GitHub, LinkedIn, email)

## License

Blog content is © Saroj. The [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme is licensed under its own terms — see the theme repository for details.
