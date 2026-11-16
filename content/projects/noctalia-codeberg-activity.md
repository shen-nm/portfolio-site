---
title: "Codeberg Activity: A Native Noctalia Shell Widget"
date: 2026-11-16
draft: false
summary: Building a privacy-first Codeberg and local Git contribution heatmap plugin for Noctalia Shell on NixOS.
tags:
  - nixos
  - noctalia
  - codeberg
  - luau
  - wayland
categories:
  - Projects
  - Desktop
showTableOfContents: true
---
After moving my repositories off GitHub and over to Codeberg, one of the few things I missed was having a quick visual pulse on my daily commit streak right in my bar. Since my desktop runs Niri with [Noctalia Shell](https://github.com/noctalia-dev/noctalia-shell), writing a native Luau plugin seemed like the cleanest way to bring a contribution heatmap into my setup without bolting on an external script or web view.

The result is **[noctalia-codeberg-activity](https://codeberg.org/shenn/noctalia-codeberg-activity)**, a bar widget and popup calendar built against Noctalia's Plugin API (`v43`).

{{< figure src="/gallery/codeberg-activity.png" alt="codeberg activity heatmap"  >}}

## Why Local Git Mode Exists

Most Git forge widgets assume all of your activity lives on a public profile, or that you're fine leaving a `read:user` API token sitting in your shell config.

For me, the repository I commit to most often is `~/nixos`. A NixOS flake is a complete blueprint of your machine, every installed package, background service, and open firewall port, so keeping that repository private is common sense. At the same time, I didn't want to manage API tokens just to count my own commits.

To solve both, the plugin supports two data sources:

- **Codeberg API:** Fetches your public contribution heatmap directly from Codeberg (with optional token support if you *do* want private remote repos included).
- **Local Git repository:** Runs `git log` asynchronously against a local path like `~/nixos` completely offline. It resolves `~/` dynamically at runtime, needs zero tokens, and never touches the network.

## Matching Noctalia's Native UI

Noctalia plugins are split into a background sync service (`sync.luau`), a bar widget (`widget.luau`), and an attached popup (`panel.luau`). Getting the data into the grid was the easy part; getting the popup to look like it shipped with Noctalia took a bit more iteration.

A few small details made the difference between "third-party box" and native shell UI:

- **Smoked-glass cards:** Instead of an opaque slab with hard divider lines, the default **Frosted cards** style uses translucent `surface/0.78` containers with subtle `outline_variant` borders so it matches the built-in Calendar and system popups. If you prefer the solid look, you can toggle back to **Classic** in the plugin settings.
- **Dynamic panel height:** Using `height = "content"` lets the popup shrink down to just the heatmap card when you hide the **Today**, **Current streak**, and **Best streak** cards, no dead space left behind.
- **True square cells:** The 53-week grid renders as `9×9` rounded squares with hover inspection for exact daily commit counts, keeping the popup compact even on smaller displays.

## Under the Hood

Noctalia's Luau environment executes plugin panels without a full POSIX shell environment, so paths and formatting needed to stay self-contained:

- **Progressive grid rendering:** Building 371 interactive cells (`53 × 7`) in one pass can cause a brief frame hitch when opening the popup, so `panel.luau` prepares rows and columns incrementally across frame ticks and caches the resulting UI tree until new data arrives.
- **Configurable everything:** From `plugin.toml`, you can switch the bar widget between icon-only or live metrics (`Today`, `Current streak`, or `Last year`), adjust the background refresh interval (`15`, `30`, or `60` minutes), and toggle every card or label in the popup individually.

## Try It Out

The source code and installation instructions are up on Codeberg:

{{< codeberg repo="shenn/noctalia-codeberg-activity" >}}

If you're already running Noctalia on Niri or another Wayland compositor and want to track either your Codeberg profile or a private local repo, clone it into your plugins directory, enable `shenn/codeberg-activity`, and point it at your username or local Git path.