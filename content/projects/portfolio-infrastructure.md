---
title: "Portfolio Infrastructure: Under The Hood Of This Web Page"
date: 2026-06-29
draft: false
summary: A rundown of the infrastructure of the page which runs on GitHub Actions and Cloudflare Pages
tags:
  - cloudflare_pages
  - github_actions
  - hugo
  - CI/CD
  - cloudflare
  - github
categories:
  - Infrastructure
showTableOfContents: true
---
## The Architecture (No FTP Allowed)

Most personal portfolios are treated like an afterthought—manually dragging files over SFTP or clicking around a clunky GUI. I wanted this space to treat infrastructure with the same discipline as a production enterprise cluster: immutable, tracked, and completely automated.

* **The Engine:** Hugo running the Blowfish theme. It compiles raw markdown into lightweight static assets in milliseconds.
* **The Edge:** Hosted entirely on Cloudflare Pages. Traffic hits Cloudflare’s global edge network instantly, giving me zero server overhead to maintain.
* **The Pipeline:** A strict Git-driven CI/CD deployment loop. I don't deploy files; I push code, and the cloud compiles the rest.

---

## Keeping the Workspace Clean (Learning from Mistakes)

When building locally on my Arch laptop, running `hugo server` compiles thousands of temporary static files under `public/` and `resources/`. Early on, I accidentally committed 16,000 lines of generated tracking junk into Git. 

To permanently shield the repository source history from compilation bloat, my `.gitignore` is locked down tight:

```text
public/
resources/
.hugo_build.lock
````

Now, the source tree stays completely pristine, tracking only the structural configuration, custom styles, and raw markdown content:

```
.
├── assets/
│   └── css/
│       └── custom.css      # Where my background & blur overrides live
├── content/                # Obsidian markdown vault syncing posts
└── config/_default/        # Core theme parameters & menus
```

## The "Don't Break Prod" Workflow

To prevent a rogue syntax typo or broken theme layout from taking down `shenmilan.com`, I implemented an isolated staging sandbox. Every feature follows a strict promotion pipeline.

### 1. The Sandbox Phase

No code or text is ever written directly on the live `main` branch. I switch over to an isolated staging workspace and fire up the local rendering engine to test changes:

```bash
git checkout staging
hugo server -D
```

### 2. Isolated Edge Testing

When I push to the staging branch, GitHub alerts Cloudflare Pages. Cloudflare spins up an isolated build container, compiles the branch independently, and generates a private preview URL (`https://staging.project.pages.dev`).

```bash
git add .
git commit -m "style: implement frosted card overlay and fixed background blur"
git push origin staging
```

If a layout script crashes or a parameter tag is malformed, the build fails safely in the container. The live production site remains completely untouched and online.

### 3. Fast-Forwarding to Production

Once the staging preview URL is visually vetted and the build is solid green, I fast-forward those verified commits right into the production branch:

```bash
git checkout main
git pull origin main
git merge staging
git push origin main
```

Cloudflare handles the final switchover atomically. The edge network swaps the assets instantly, rolling out the new features to the world with zero downtime.

## Infrastructure Retrospective

- **Atomic Safety:** Treat your portfolio like production infrastructure. If a deployment fails, nobody should ever see a `404` or a broken page.
    
- **Conventional Commits:** Prefixing messages with labels like `feat:`, `style:`, or `chore:` might seem tedious at first, but it makes scanning `git log --oneline` incredibly clean and signals disciplined engineering.