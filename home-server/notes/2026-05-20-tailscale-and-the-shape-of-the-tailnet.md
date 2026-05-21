# Tailscale, and the shape of the tailnet

**Date:** 2026-05-20
**Project phase:** 5 (remote access)

## Context

The home server works fine on the home network. The question Phase 5 addresses: how do I reach it from anywhere — coffee shop, work, on cellular, anywhere — without exposing services to the public internet?

The traditional answers all have real downsides:
- **Port forwarding** on the home router exposes services to the public internet. You're then one zero-day vulnerability away from being a node in someone's botnet.
- **Dynamic DNS** solves the "my home IP changes" problem but still requires port forwarding.
- **Self-hosted VPN** (OpenVPN, WireGuard from scratch) is the right idea but a lot of configuration.

Tailscale is a managed WireGuard VPN that solves the configuration headache. Free for personal use up to 100 devices. After Phase 5: my phone reaches the server from cellular as if I were home.

## What Tailscale actually is

It's worth being precise about this, because the marketing language ("zero-config", "magic") hides what's actually happening, and engineers should know the actual mechanism.

Three things are stitched together:

**1. A coordination server** that Tailscale runs. When a device joins your tailnet, it registers with this server. The server's job is to keep a directory of "which devices belong to which user, and how can they reach each other right now."

**2. WireGuard tunnels** between pairs of devices that want to communicate. WireGuard is an open-source, modern VPN protocol — much simpler and faster than OpenVPN. Each tunnel is point-to-point and end-to-end encrypted; the coordination server doesn't see traffic.

**3. NAT traversal** — the trick that makes this work without port forwarding. Most home networks sit behind NAT (network address translation), which lets outbound connections work but blocks inbound ones. Tailscale uses a technique called UDP hole punching: both devices send outbound packets to each other simultaneously, and the NAT routers — having seen the outbound — accept the matching inbound packets as "responses." Effectively, the two devices punch a hole through their respective NATs and meet in the middle.

When direct NAT traversal fails (about 10% of the time, depending on network type), Tailscale falls back to a relay server called DERP. Traffic still goes through encrypted tunnels; just a third party briefly forwards it.

## What this looks like from inside

After installing Tailscale on the laptop and the phone, each device gets a private IP in the `100.x.x.x` range. These are addresses that only exist inside my tailnet — they're not routable on the public internet at all.

When my phone (on cellular, anywhere on Earth) opens `http://homeserver:8096`:

1. **DNS resolution.** "What IP is `homeserver`?" Tailscale's MagicDNS — which intercepts DNS for `*.ts.net` and bare device names — answers with the laptop's tailnet IP, e.g. `100.101.93.110`.
2. **Routing.** The phone tries to send packets to `100.101.93.110`. The Tailscale client on the phone catches these, encrypts them, wraps them in a WireGuard packet, and sends them through whatever the phone's normal internet connection is.
3. **Coordination.** If this is the first conversation with the laptop in a while, the coordination server is consulted briefly to learn current addresses.
4. **NAT traversal.** Hole punching establishes a direct UDP connection between phone and laptop, wherever they each are.
5. **Delivery.** Laptop's Tailscale client unwraps the packet, sees an HTTP request for port 8096, hands it to Jellyfin.

All of this in under 200ms.

## What went wrong, and what it taught me

**Tailscale wasn't enabled to start at boot.** I ran `sudo tailscale up` to bring the daemon up, which connected the server to the tailnet *for that session*. But I didn't run `sudo systemctl enable tailscaled`, which would have configured the service to autostart on boot.

I discovered this when the laptop apparently suspended itself overnight (a separate bug), came back up automatically, but didn't reconnect to the tailnet. The home network worked fine; the tailnet showed the server as offline. From outside the home network, the server was unreachable.

**Fix:** `sudo systemctl enable tailscaled` makes the daemon start on every boot. Now the server stays on the tailnet across reboots.

**What I learned:** there's a real distinction between "this is running right now" and "this will be running after the next reboot." `systemd` services have both a current state (active/inactive) and an enablement state (enabled/disabled at boot). On Linux you need to verify both, especially for services you intend to run permanently. The lesson generalizes — any time I start a long-running service, the next step is asking "and is it set to come back automatically?"

## A small bonus discovery

I'd assumed the only way to access the tailnet from devices that can't install the Tailscale client (e.g. my school's locked-down Chromebook) was to use the rclone-via-Google-Drive workaround we'd discussed.

It turns out `login.tailscale.com` itself loads fine on the locked-down Chromebook — the admin console isn't a tailnet-internal site, just the management dashboard. That doesn't get me into the actual services (those still need a client on the device), but it does give me **Taildrop** in a browser, which can transfer files into the tailnet from any web-capable device. Not a full solution, but a real capability I didn't expect to have.

The broader lesson: the boundaries of what's "possible" with a service are often softer than you assume. A 5-minute test on the locked-down device showed me a capability I'd already given up on.

## What this exercise was actually for

The technical content — install Tailscale, sign in on two devices, demonstrate it works from cellular — is again maybe an hour of work. The value was in:

1. Recognizing that "magic" technologies have specific mechanisms underneath, and forcing myself to articulate them
2. Internalizing the `is-it-running` vs `is-it-set-to-start` distinction for long-running services, which I'll now check by default
3. Discovering the Taildrop-via-browser side channel by actually testing assumptions instead of trusting them
