# Home Server & Network Lab

A Linux-based home server and network lab built on repurposed hardware (a 2011 Gateway laptop, Intel i3-2310M, 4 GB RAM). The goal: build genuine hands-on experience with the Linux/Docker/networking stack that underlies most modern engineering infrastructure, with a working set of self-hosted services as the deliverable.

## What it does

- **Media server** (Jellyfin) — streams video and music to phones, TVs, and browsers on the network and remotely
- **Network-wide ad blocking** (Pi-hole) — intercepts ad/tracker DNS for every device on the network
- **Remote access** (Tailscale) — secure access to the server from anywhere without exposing ports to the public internet
- **Container management** (Portainer) — web UI for managing all of the above
- **Backups, firewall, automatic security updates** — boring but essential

All services run in Docker containers for isolation, reproducibility, and clean upgrades. Configuration lives in version-controllable compose files under a single directory.

## Architecture
[Phone/TV/Laptop] ──> [Spectrum Router] ──> [Pi-hole DNS] ──> Internet
                                       │
                                       └──> [Jellyfin, Portainer, ...]
                                              (Docker on Gateway laptop)

[Remote phone/laptop] ──Tailscale tunnel──> [Same Gateway laptop]

## Status

**In active development.** Phases 1–5 complete; remaining phases (storage, couch interface, emulation, file sharing) require additional hardware purchases scheduled across the next 1–2 months.

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | Debian 13 install (legacy BIOS, manual partitioning) | ✓ Complete |
| 2 | SSH hardening, static IP, security baseline | ✓ Complete |
| 3 | Docker + Jellyfin + Portainer | ✓ Complete |
| 4 | Pi-hole network-wide ad blocking | ✓ Complete (pending router DNS) |
| 5 | Tailscale remote access | ✓ Complete |
| 6 | Automated backup strategy | Planned |
| 7 | External storage drives | Planned |
| 8 | TV/HDMI/audio output, Bluetooth controller | Planned |
| 9 | Retro emulation (RetroArch + ES-DE) | Planned |
| 10 | Web-based file sharing | Planned |

## Technical skills exercised

- **Linux system administration** — Debian install on legacy hardware, systemd service management, journalctl debugging, fstab, networking via NetworkManager/nmcli
- **Docker** — multi-container deployment via compose, volume management, network modes (bridge vs host), permission handling across container/host boundaries
- **Networking** — static IP assignment, DNS architecture, port management, firewall (ufw) rule design, VPN tunneling (WireGuard via Tailscale)
- **Security** — SSH key auth, principle-of-least-privilege firewall rules, automatic security patching, password hygiene
- **Debugging** — reading kernel logs, diagnosing failed services, working through BIOS-level issues during install

## Documentation structure

- **This README** — overview, status, what was built
- **`notes/`** — dated lab notebook entries capturing the engineering process, including dead-ends, surprises, and decisions. These are written candidly — they're closer to a working engineer's notebook than to a finished writeup.

## Hardware

| Component | Spec |
|-----------|------|
| Laptop | Gateway NV57H, ~2011 |
| CPU | Intel i3-2310M (2 cores, 4 threads, 2.1 GHz) |
| RAM | 4 GB DDR3 |
| Storage | 500 GB WD HDD (internal) |
| Network | Gigabit Ethernet, Cat6 to router |
| BIOS | Legacy / MBR only (no UEFI) — relevant for install |

The choice to use this specific laptop was deliberate: working within real hardware constraints (limited RAM, no UEFI, no virtualization features) forces decisions about what services are actually appropriate, rather than throwing capacity at problems.

## Why this project
The Linux/Docker/networking stack underlies a substantial fraction of modern engineering infrastructure — from embedded edge devices to industrial control systems to test/measurement equipment. Hands-on experience with this stack is a complement to electrical engineering coursework, not a substitute for it. The aviation industry in particular runs heavily on Linux-based systems for ground equipment, data analysis pipelines, and increasingly the avionics themselves; familiarity with the stack is functional.
It also has the advantage of producing a working artifact: the server runs, the services work, friends and family use it. Engineering projects that produce nothing observable at the end are harder to learn from.
