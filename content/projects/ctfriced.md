---
title: "CTFRiced"
tagline: "Ricing CTFd"
category: "CTF Tools"
status: "stable"
type: "platform"
description: "A customised CTFd: anti-cheat, dockerised challenges, geo challenges and a theme that doesn't look like 2015."
language: "Python"
stars: 12
forks: 1
tech:
  - "CTFd"
  - "Python"
  - "Docker"
  - "MariaDB"
  - "Redis"
  - "Jinja2"
github: "https://github.com/AbuCTF/CTFRiced"
demo: "https://ctf.h7tex.com"
draft: false
---

The CTFd we actually run events on. Ricing CTFd to the best of my abilities.

## Versions

Tagged per event, because a live CTF is not the time to find out a plugin broke.

- **3.7.7**: fully tested. Multiple plugin support, custom UI, docker challenges, anti-cheat.
- **3.8.0**: added the discord notifier, geoint support, the fractals theme.
- **3.8.4**: rebased onto latest CTFd, UI inconsistencies fixed, four security patches.

## Anti-cheat

The plugin most worth talking about. Four detectors, run together:

**Duplicate flags.** Two teams submitting a flag that should be per-team.

**Brute force.** More than ten submissions in a 60-second window.

**IP sharing.** Three or more accounts from one address.

**Submission sequence similarity.** The interesting one. Two teams solving the same challenges in the same order isn't suspicious on its own. A good team and a mediocre one both start with the easy web challenge. So the detector compares two things with `difflib.SequenceMatcher`: the raw solve sequence, and the *time deltas between solves*, normalised to minutes.

Matching order is weak evidence. Matching rhythm is not. If one team's solve gaps track another's within 30 seconds, they aren't solving, they're relaying.

Auto-ban is off by default. It writes alerts and lets a human decide.

## Docker challenges

Multi-server with load balancing, containers provisioned per team and cleaned up after, domain mapping so players get a real hostname instead of an IP and port, health monitoring with failover.

The CTFd container gets the host Docker socket mounted in, which is exactly as much trust as it sounds like and the reason the whole stack sits behind nginx on an `internal: true` compose network.

## Geo challenges

Solve by clicking a point on a Leaflet map. Correct if you're within the challenge's tolerance. Good for OSINT categories where the answer is a place and not a string.

## Notifier

Extended from the upstream ctfd-notifier so it fires on second and third blood, not just first. Discord, Twitter and Telegram.

## Theming without a rebuild

The custom UI ships as two files you paste into `/admin/config`: `theme-header.css` and `theme-footer.js`. No theme rebuild, no container restart, and it survives a CTFd upgrade.

The CSS is a `:root` palette, dark by default with a light override, accent `#64ffda`. The JS is a canvas overlay at `z-index: -3` running particles, a bezier wave and a proximity-connection grid at 60 FPS, behind everything else on the page.

## Stack

CTFd 3.8.4, nginx, MariaDB 10.11, Redis. Four themes in tree: admin, core, core-deprecated, and fractals, which self-hosts its Inter, JetBrains Mono and Space Grotesk subsets rather than calling out to Google Fonts.

Infrastructure writeups for the events this ran: [H7CTF 2024](/docs/dev/h7ctf24-infrastructure/).
