---
title: Power Outage Post-Mortem & Tailscale/Cloudflared Race Condition
date: 2026-06-30
draft: false
summary:
tags:
  - homelab
  - networking
  - systemd
  - tailscale
  - cloudflare
categories:
  - engineering_logs
showTableOfContents: true
---
## The Incident

At approximately 14:15 local time, a brief grid failure caused a hard reboot of my primary home-lab node. While the hardware physically survived the power cycle and booted back up cleanly, the public-facing portfolio went completely dark.

Internal routing was functional, but the site was throwing a `502 Bad Gateway` at the edge network.

---

## The Root Cause: Systemd Dependency Race

Upon digging into the system logs via SSH, I discovered a classic race condition between my network mesh overlay and my reverse proxy tunnel. 

The `cloudflared` daemon was configured to bind explicitly to an internal backend service routing over my private Tailscale interface. During the boot sequence, systemd initiated both services simultaneously. 

Because `cloudflared` initialized milliseconds before `tailscale` could complete its handshakes and instantiate the virtual interface, the tunnel couldn't resolve the route, panicked, and crashed.

---

## The Fix (Enforcing Order)

To fix this permanently, I had to drop into the systemd configuration on the host and explicitly instruct the Linux kernel not to touch the Cloudflare tunnel until the Tailscale service is fully online and stable.

### 1. Edit the Cloudflared service file overrides
```bash
sudo systemctl edit cloudflared
```

### 2. Inject the dependency constraints

Inside the systemd override configuration, I appended the explicit ordering rules under the Unit block:

```
[Unit]
Requires=tailscaled.service
After=tailscaled.service network-online.target
```

### 3. Reload the systemd daemon and test the boot ordering

```bash
sudo systemctl daemon-reload
sudo systemctl restart cloudflared
```

## Infrastructure Retrospective

- **Network Dependency:** Never assume local network interfaces are instantly available on boot. If a service relies on an overlay network or a VPN mesh, the systemd unit file _must_ explicitly state `After=vpn-service.service`.
    
- **Resilience Testing:** A true high-availability system should be able to recover gracefully from a cold boot without human intervention. Fixing this race condition means the lab can now survive sudden blackouts completely hands-off.
