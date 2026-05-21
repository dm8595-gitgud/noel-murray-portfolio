# SSH, then Docker — finding the pattern

**Date:** 2026-05-15
**Project phases:** 2 and 3 (SSH hardening, Docker + Jellyfin)

## Context

With Debian installed, two things needed to happen before the project could really begin: get remote access working so I could stop using the laptop's broken keyboard, and set up Docker so services could be deployed cleanly.

What was interesting was recognizing the *reusable pattern* that emerged on the Docker side, which has shaped everything since.

## SSH first 

Standard SSH setup, but with a few deliberate choices:

- **Key-based authentication** generated on the Windows side, public key copied to the server. Password auth still enabled — I didn't disable it, because locking yourself out of your own server with a broken laptop keyboard would be a bad day. Keys are *additional* security, not exclusive.
- **Static IP via NetworkManager's `nmcli`** rather than editing config files directly. Debian 13 ships with NetworkManager as default; `nmcli con mod` is the documented way to make IP changes persistent. Trying to edit `/etc/network/interfaces` (the older approach) doesn't work because NetworkManager owns the interface and overrides whatever's there.
- **Lid-close behavior** changed so closing the laptop doesn't suspend it. The server lives in a TV cabinet eventually; it must stay running when the lid is shut. Setting `HandleLidSwitch=ignore` in `/etc/systemd/logind.conf` does this, but later turned out to be insufficient (more on that in a future entry — the laptop suspended on its own from idle anyway, taking the server offline for ~13 hours before I noticed).

## Docker — and the pattern I now use for everything

Docker itself was straightforward to install (vendor's official apt repo, four packages, done). The interesting realization came on the second container.

The first service deployed was **Portainer** — a web UI for managing Docker. The setup was:

1. Make a folder: `/srv/docker/portainer/`
2. Inside it, write a `compose.yml` file describing the container
3. Run `docker compose up -d`

The second service was **Jellyfin**, the media server. Setup was:

1. Make a folder: `/srv/docker/jellyfin/`
2. Inside it, write a `compose.yml` file describing the container
3. Run `docker compose up -d`

The same pattern. Every service that's been added since (Pi-hole, eventually Filebrowser, eventually code-server) uses the same three steps. The folder name changes, the contents of `compose.yml` change, but the pattern doesn't.

This is the kind of thing that doesn't show up in a tutorial — tutorials show you "how to install X" service by service, as if each is a special case. The realization that *they all follow the same shape* makes everything after the second one significantly faster, and makes the system easier to reason about as a whole.

## The architectural side of this

Beyond the convenience, the pattern has real engineering consequences:

- **All service state lives under `/srv/docker/`.** Backing up the whole server reduces to: rsync `/srv/docker/` somewhere safe.
- **Each service is isolated.** Jellyfin can't accidentally damage Portainer. A misbehaving container is restarted, not surgically repaired.
- **Each service is reproducible.** The `compose.yml` is the entire definition. To reinstall the system from scratch: install Docker, copy the compose files back, `docker compose up -d` in each folder. ~10 minutes from blank OS to all services running.
- **Each service is upgradable in place.** `docker compose pull && docker compose up -d` fetches the latest image and restarts. No "this version conflicts with the system Python" headaches that I've run into before running linux in server mode.
- I realize this isn't exactly rocket surgery, and that most server people probably realize all of these benefits already, but to me, this is so far superior to the ways I've installed things previously, that I'm very excited at the moment. Especially in linux, so many issues have come from files being sent all over the spread out filesystem, mismatching files with updates, installing the wrong versions, etc. The container system is very exciting. 

## Permissions: the one part that bites

Docker containers run as a specific user inside the container, which may or may not match the user on the host machine. If a container writes a file to a mounted volume as user 1000, and your host user is user 1001, you can't edit that file from the host without sudo.

**The fix**, which I now do by default for every container:
- Run `id` on the host to find my user's uid/gid (1000:1000 in my case)
- Add `user: 1000:1000` to the container's compose.yml

This makes the container write files as me, so files in mounted volumes are owned by me on the host, and everything stays editable without permission gymnastics.

## What I would do differently

In hindsight, I'd have set up the firewall (ufw) and unattended-upgrades **at the same time** as the rest of Phase 2, rather than as a Phase 2.5 cleanup pass later. They're not optional, and doing them later meant I had a window of days where the server was running with everything exposed and no auto-patching. Not catastrophic — the firewall did get added — but it's the kind of thing where "I'll get to it" is a real failure mode in security work.

## What this exercise was actually for

The technical content of Phase 3 — install Docker, run Jellyfin — is genuinely a 1-hour task. The value of doing it was learning to see the *pattern*: every service that comes after benefits from the consistent structure. That generalization-from-examples skill is what I want practice in, more than knowing any single tool.
