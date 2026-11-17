---
title: Push-to-Deploy Homelab Configs on Proxmox with Forgejo Actions
date: 2026-11-17
draft: true
summary: Replacing manual SSH config edits across 12 Proxmox LXCs with a push-to-deploy setup using Codeberg, Forgejo Actions, and pct push.
tags:
  - proxmox
  - gitops
  - homelab
  - forgejo
  - lxc
  - CI/CD
  - workflow
categories:
  - Homelab
showTableOfContents: true
---
Managing config files inside one Linux container is easy. Managing them across a dozen Proxmox LXCs, running things like Homepage, Navidrome, Cloudflared, and Nginx Proxy Manager, gets messy fast.

For a while, updating a dashboard link or tweaking a reverse proxy rule meant SSHing into whichever container ran it, editing YAML in place, and hoping I'd remember what I changed three months later. I wanted a proper `git push` workflow instead: edit everything locally on my NixOS machine, push to Codeberg, and let the changes land in the right container automatically.

Here is how I set that up using **Codeberg**, **Forgejo Actions**, and Proxmox's `pct` CLI, without opening any inbound ports or installing agents inside every container.

## Ground Rules

Before writing any workflow YAML, I wanted a few constraints in place:

- **No inbound firewall ports:** My homelab sits behind NAT, Tailscale, and Cloudflare Tunnels. Codeberg shouldn't need direct webhook access into my network.
- **Nothing extra inside the LXCs:** I didn't want to install `git`, SSH keys, or runner daemons inside 12 separate containers.
- **Per-container deploys:** Editing my Homepage bookmarks shouldn't touch or restart Navidrome or anything else.
- **Local-only secrets:** `.env` files and API tokens stay on the homelab and never touch a remote Git repo.

## How It Fits Together

Instead of putting a runner inside each container, I put **one runner directly on the Proxmox host** and tied it to a single private monorepo (`homelab-infra`) on Codeberg:

```text
┌───────────────────┐     git push      ┌──────────────────────────┐
│  NixOS Workstation│ ────────────────► │  Codeberg (homelab-infra)│
└───────────────────┘                   └──────────────────────────┘
                                                      ▲
                                                      │ Outbound HTTPS
                                                      │ (Long-Poll)
                                        ┌──────────────────────────┐
                                        │  Proxmox VE Host         │
                                        │  └─ forgejo-runner       │
                                        └──────────────────────────┘
                                           │         │         │
                                    pct push  pct push  pct exec
                                           ▼         ▼         ▼
                                       [CT 102]  [CT 106]  [CT 108]
                                      Navidrome Cloudflared Homepage
````

### 1. Monorepo Layout

Every container gets its own top-level folder prefixed with its Proxmox VMID, which keeps the mapping between Git paths and target containers obvious at a glance:

```text
homelab-infra/
├── .forgejo/
│   └── workflows/
│       └── deploy.yml
├── .gitignore          # Excludes *.env and local secrets
├── 102-navidrome/
│   └── docker-compose.yml
├── 106-cloudflared/
│   └── config.yml
└── 108-homepage/
    ├── bookmarks.yaml
    ├── services.yaml
    ├── settings.yaml
    └── widgets.yaml
```

### 2. Outbound-Polling Host Runner

Because Codeberg runs on Forgejo, it supports self-hosted runners via `forgejo-runner`. Rather than listening for incoming webhooks, the daemon holds an outbound HTTPS connection to `codeberg.org` and polls for queued jobs.

Registering it with the `self-hosted:host` label tells Forgejo to run workflow steps directly on the Proxmox host shell instead of spinning up nested Docker containers:

```YAML
server:
  connections:
    codeberg:
      url: https://codeberg.org/
      uuid: <RUNNER_UUID>
      token: <RUNNER_TOKEN>

runner:
  labels:
    - "self-hosted:host"
```

### 3. Path-Filtered Workflow and `pct push`

Since the runner executes right on the hypervisor, it already has access to Proxmox's `pct` tool and can push files straight into a container's filesystem without SSH or `rsync`:
```YAML
name: Deploy Homelab LXCs

on:
  push:
    branches: [ main ]
    paths:
      - '108-homepage/**'

jobs:
  deploy-108-homepage:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - name: Push Homepage config to CT 108
        run: |
          pct status 108 | grep -q running || exit 0
          for file in 108-homepage/*; do
            [ -f "$file" ] || continue
            pct push 108 "$file" "/opt/homepage/config/$(basename "$file")"
          done
```

A couple of small details keep this from breaking in day-to-day use:

- **Top-level `paths:` filters:** Forgejo checks `paths:` at the trigger level, so a commit to one container's folder skips every other job completely.

- **Stopped-container check:** A few LXCs in my lab are only booted on demand. Checking `pct status <VMID> | grep -q running` lets updates for powered-off containers exit cleanly instead of marking the run failed.

- **Post-deploy restarts when needed:** Services like Homepage hot-reload YAML changes as soon as `pct push` writes them. For Docker Compose stacks like CT 102, a follow-up `pct exec 102 -- docker compose up -d` applies the updated compose file automatically.

## Gotchas on Bare-Metal Proxmox

Getting the runner working smoothly on the Proxmox host came with three small surprises:


1. **Forgejo Runner v7+ config format:** Older guides still reference `forgejo-runner register` with a single registration token. Current Codeberg instances expect a UUID + Token pair under `server.connections` in `config.yml`, which requires `forgejo-runner` v7 or newer (I used `v12.8.2`).

2. **Array checks in workflow `if:` expressions:** Trying `if: contains(github.event.head_commit.modified, '108-homepage/')` silently skipped every job because Forgejo passes `modified` as a JSON array of exact file paths rather than a string. Using top-level `paths:` triggers avoided the issue entirely.

3. **Host dependencies for `actions/checkout`:** Minimal Proxmox VE installs don't include `git` or `nodejs` out of the box. Because `actions/checkout@v4` runs a Node script when executing in `:host` mode, both need to be installed via `apt` on the hypervisor first.

## Trade-Offs

Running a CI runner straight on the Proxmox host obviously trades container isolation for simplicity—I wouldn't do this on a shared repo or anywhere untrusted PRs can trigger workflows. For a private, single-user repo holding only my own config files, though, it's a trade-off I'm completely fine with: full Git history, zero open ports, and single-command deploys from my workstation.