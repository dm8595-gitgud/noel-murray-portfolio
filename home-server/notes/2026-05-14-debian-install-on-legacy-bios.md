# Debian install on a legacy-BIOS laptop

**Date:** 2026-05-14
**Project phase:** 1 (initial install)

## Context

Starting fresh: 2011 Gateway laptop, wiping whatever was on it. Plan was a standard Debian 13 install. I expected this to be the easy part of the project — pop in a USB, click through, done- found some issues that were more challenging than expected because the laptop was very old. 

## What I ran into

### 1. UEFI vs legacy BIOS

First USB installer wouldn't boot. Used Rufus on Windows with default settings, which produces a UEFI-only image. This laptop is old enough that its firmware is **legacy BIOS only** — no UEFI support at all. The installer USB was invisible to the boot menu.

**Fix:** recreated the USB in Rufus with the partition scheme explicitly set to MBR and the target system to BIOS (or UEFI-CSM). Booted on first try after that.

**What I learned:** UEFI became standard around 2012. Anything older is legacy BIOS, which constrains how you create install media, how you partition disks, and how the bootloader is installed. Worth checking firmware era before assuming modern defaults.

### 2. Old LVM volumes blocking the installer

Debian's partitioner couldn't proceed. It saw old LVM (Logical Volume Manager) volumes from a previous Linux install attempt on this drive and refused to do anything until they were cleared.

**Fix:** dropped to the installer's shell, used `vgremove` and `pvremove` to clear the LVM signatures off the disk, then went back to the partitioner. Once the volumes were gone, partitioning proceeded.

**What I learned:** LVM is a layer on top of physical partitions that creates a virtual disk pool. When you remove a Linux install that used LVM, the LVM metadata can persist on the disk even if you delete the partitions. New installers see the metadata and refuse to overwrite it as a safety precaution.

### 3. GPT vs MBR partition tables on a BIOS system

I chose GPT (the modern default) for the partition table without thinking. Then GRUB — the bootloader — refused to install, complaining it couldn't find a place to write itself.

The issue: on a UEFI system, the firmware itself knows how to read GPT and find the bootloader from an EFI System Partition. On a **legacy BIOS** system, the firmware reads the first sector of the disk (the MBR), which expects a 446-byte stub. GPT reserves that area but doesn't include the stub by default. So GRUB had nowhere to put itself.

**Fix:** created a small (~8 MB) unformatted partition at the start of the disk, flagged as `bios_grub`. This is GPT's official escape hatch for legacy BIOS compatibility — a reserved region GRUB knows how to use as its first-stage bootstrap. After adding this partition, GRUB installed cleanly.

**What I learned:** partition table format and firmware type need to match. GPT + UEFI works natively. MBR + legacy BIOS works natively. GPT + legacy BIOS works, but requires the explicit `bios_grub` partition. The Debian installer's defaults assume modern hardware, and on old hardware the diagnostics aren't always clear about what's wrong.

## Diagnostic moves that helped

- Reading the installer's log on tty4 (Ctrl+Alt+F4 to switch consoles) showed the underlying error messages that the GUI was hiding.
- When stuck, dropping to a shell and using `lsblk -f`, `parted -l`, and `wipefs -a` made disk state visible. The GUI doesn't show LVM metadata that's still there; the CLI tools do.
- Searching error messages verbatim — not paraphrased — found the relevant Debian wiki pages and forum threads. The bios_grub partition fix came from there.

## Time taken

About 4 hours (with time in there to take care of the baby-so not too bad!) from "I'll just install Debian" to "I have a working desktop." Most of it was the BIOS / partitioning detour. If I were doing this again on the same hardware: I'd start with a Rufus install media set explicitly to MBR/BIOS, GPT with bios_grub partition pre-planned, and wipe LVM signatures before letting the installer touch the disk. The actual Debian install itself takes about 20 minutes.

## What this exercise was actually for

The point of doing this on old hardware was specifically to encounter problems like these, and still find a way to make this into a nice server. I suppose I could simply purchase a complete and functioning server too! That's not the point- I would have learned nothing. 
