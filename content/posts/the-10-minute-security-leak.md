---
title: "The 10-Minute Security Leak: A Lesson in Git History & Key Rotation"
date: 2026-06-28
draft: false
tags: ["Git", "Security", "Proxmox", "Homelab"]
---

### The Situation
While migrating my container configuration files to local environment variables (`.env`), a spacing syntax error caused my application container diagnostics logs to output my live Proxmox hypervisor API tokens in plain text to a public GitHub repository.

### The Mitigation Pipeline
Instead of panicking, I executed an immediate triage protocol:
1. **Repository Purge:** Because Git maintains a permanent immutable history, simply deleting the file wasn't enough. I locally purged the tracking cache tree using `git rm -r --cached .` and force-pushed a wiped timeline to permanently overwrite the exposed commits on the remote server.
2. **Token Revocation:** I immediately opened my Proxmox cluster permissions dashboard, destroyed the compromised tokens, and regenerated a completely new alphanumeric security string.
3. **Tracking Rules Enforcement:** I appended relative path constraints (`config/logs/`) to my `.gitignore` to ensure background server files are permanently ignored by the staging index.

### Key Takeaway
Good systems management isn't about never making a formatting mistake; it's about building a fast internal incident response model to protect core infrastructure.