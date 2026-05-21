# The server went to sleep

**Date:** 2026-05-21
**Project phase:** post-5 (operational incident)

## What happened

I was at work, away from the home network, and tried to SSH to the server through Tailscale. Connection timed out.

I opened the Tailscale app on my phone. My phone showed online (green dot). The server (`homeserver`) showed offline (gray dot). Whatever was wrong, Tailscale itself wasn't the problem — the server simply wasn't on the tailnet anymore.

Couldn't do anything about it from work. Got home, found the laptop unresponsive — screen black, no response to keyboard. Held the power button to force a shutdown, then powered it back on. Booted normally.

## Diagnosis

Once back in over SSH, I went looking for what happened. The relevant tool is `journalctl`, which exposes the systemd log — the comprehensive record of everything the system has done across boots.

```
journalctl --list-boots
```

This lists past boot sessions, indexed from 0 (current) backwards. The one *before* the current boot is the session that died.

```
sudo journalctl -b -1 -n 50 --no-pager
```

This dumps the last 50 log lines from the previous boot — the moments before things went dark.

The logs were boring. The last entries were a routine cron job at 04:17 AM, followed by silence until I rebooted around 5:50 PM. **Thirteen and a half hours of nothing.** No kernel panic. No thermal warning. No graceful shutdown message. Just an abrupt stop.

"Logs stop cleanly with no error" almost always means the kernel itself wasn't running anymore — either a crash with no time to write to disk, or, much more likely, **suspend**. The system put itself to sleep and never woke up.

## The deeper problem

I thought I'd disabled suspend already. In Phase 2 I'd changed `/etc/systemd/logind.conf` to ignore the lid switch — so closing the laptop wouldn't trigger suspend. That part worked.

What I hadn't disabled was **idle-driven suspend**. systemd's power management has multiple suspend triggers:
- Lid close → handled by `logind.conf`
- Power button → also `logind.conf`
- **Idle timeout** → handled separately, can fire even with the lid open

At 04:17 AM, with the system quiet, an idle timer apparently fired and the laptop suspended itself. The lid being open or closed had nothing to do with it.

## The fix

`systemd` has four "sleep targets" — abstract goals the system can pursue: `sleep.target`, `suspend.target`, `hibernate.target`, and `hybrid-sleep.target`. Anything that wants to suspend the system has to activate one of these targets to do so.

You can **mask** a target, which makes it unreachable — any attempt to activate it silently fails. For a server, you mask all four:

```
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Now suspend is mechanically impossible. The idle timer can fire as much as it wants; it has nothing to activate.

While I was in there, I also ran:

```
sudo systemctl enable tailscaled
```

because the previous boot had shown me that Tailscale wasn't set to autostart — which is what made the outage so dramatic in the first place. The server was down *and* invisible. With autostart, it would have come back online and silently announced itself on the tailnet, and I'd never have noticed from work.

## What this taught me

A few things, in rough order of importance:

**1. "Disable feature X" usually has multiple implementations to disable.** I disabled one path to suspend (lid close) and assumed I'd disabled suspend. Modern Linux power management has at least three independent paths to the same destination. For a server, you have to mask the destination itself, not block individual paths to it.

**2. The hard part of a remote system is the case where it's neither up nor responding to diagnostics.** A server that's working tells you it's working. A server that's broken in a known way tells you what's broken. A server that's silently sleeping looks identical to "the building lost power" or "the network cable came loose" or "the SSD failed." You can't troubleshoot what you can't reach.

The defense against this is making remote unreachability **less probable**, since it's hard to make it more diagnosable. That means: don't run a server on hardware with aggressive power management. Don't rely on services that depend on each other in a chain. Don't depend on physical access to recover.

**3. Logs that stop cleanly with no message are themselves a diagnostic.** The absence of error is information. I almost overlooked it because I was expecting to see a crash.

**4. Investigate incidents fully, even when the immediate fix is fast.** "Hold power, reboot, it works" was the 30-second fix. The actual *root cause* — idle suspend on a server — took 20 more minutes to identify and would have recurred indefinitely if I'd stopped at the reboot. Skipping the root cause is a form of technical debt that compounds.

## What this exercise was actually for

The whole project is about learning the operational side of running systems, not just the install side. Real systems break in weird ways. This is the kind of debugging that doesn't get into tutorials because it's idiosyncratic, but it's the bulk of the actual work of keeping things running.

The good outcome of this incident isn't that the server now stays up — it's that I have a procedure now: when something is silently unreachable, the first move is `journalctl --list-boots` followed by `journalctl -b -1`. That procedure will be useful many more times.
