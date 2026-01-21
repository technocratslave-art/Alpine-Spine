# Alpine-Spine
Immutable Alpine-based host with a locked boot spine and bay0 PID-1 warden. The base never changes. All functionality lives in signed, disposable pods (personas) with strict resource ceilings, network isolation, and watchdog enforcement. Shells and embodiments define UX and price tiers without touching the core.

Below is a final-pass README written as if it’s about to be handed to another senior engineer or auditor. No options, no forks, no marketing fluff—just the system as it is.


Alpine Spine / Bay0

What This Is

This repository defines an immutable Alpine Linux host (“the Spine”) with a PID-1 warden (“bay0”) that owns the hardware and enforces all invariants.
The base system never changes at runtime. There is no package manager, no upgrades in place, and no recovery shell.

All user-facing functionality runs inside signed, disposable personas (pods) that are strictly isolated, resource-bounded, and replaceable.

If something misbehaves, it is killed or the system reboots.
There is no negotiation.


Core Principles

Immutability first
The Spine is a dm-verity–verified squashfs. Read-only by construction.

No recovery paths
Boot is A → B (last known good) → power off. No shell. No menu.

PID-1 authority
bay0 is the only long-lived process. If it wedges, the kernel watchdog reboots the system.

Containment over cleverness
Personas are isolated with namespaces, cgroup v2 ceilings, seccomp, minimal /dev, and explicit network policy.

Replace, don’t repair
Personas are disposable. If compromised, purge and recreate. Persistence lives only in /persist, never in the Spine.



Architecture Overview

Firmware / Bootloader
        ↓
Kernel + initramfs
        ↓
Spine (squashfs, dm-verity, read-only)
        ↓
bay0 (PID 1, warden)
        ↓
Personas (signed pods)

The Spine (Base Layer)

Alpine Linux

SquashFS + dm-verity

No writable root

No runtime updates

No shell on boot failure

No user interaction


bay0 (Warden)

Runs as PID 1

Enforces invariants at boot:

Root is read-only

dm-verity active

/persist mounted nodev,nosuid,noexec


Opens and tickles /dev/watchdog

Launches and purges personas

Owns cgroups, namespaces, network policy

Reboots the machine on containment failure


Personas (Pods)

Signed squashfs images

Must appear in a signed inventory

Launched via double-fork PID namespace

Have:

Explicit CPU, memory, PID ceilings

Minimal /dev (tmpfs + mknod)

Explicit network mode (or none)

Seccomp-filtered syscalls


Can be purged at any time

Never modify the Spine


Persistence Model

/persist is the only writable disk

Mounted with: nodev,nosuid,noexec

Personas only see explicit bind mounts

Exec is restored only on the bind mount, never on /persist itself

Losing /persist means data loss by design


Boot & Update Model

Two Spine slots: A (active) and B (last known good)

On boot:

Try selected slot

Fallback to the other

If both fail → power off


Updates are offline:

Write new Spine to inactive slot

Reboot

After stable uptime, mark “good”


No in-place upgrades

No rollback protection in v0.1 (documented, intentional)



Threat Model (v0.1)

Defended against:

Remote compromise

Persistence across reboot

Lateral movement between personas

Accidental damage from user software


Not defended against:

Physical attacker modifying the boot partition

Hardware-level attacks

This is explicit and documented. Secure Boot + TPM are v0.2 concerns.


What This Is Not

Not a general-purpose distro

Not a container platform

Not a VM manager

Not “secure by policy”

Not forgiving


This system assumes failure is normal and handles it by reset, not recovery.


Repo Structure (High Level)

/initramfs        # Early boot, slot selection, LUKS unlock
/spine            # Build scripts for immutable base
/bay0             # PID-1 warden (Rust)
/docs
  /constitution   # Rules that must never drift
  /implementation # How those rules are enforced
/tools            # Signing, image creation, installers

Constitutional documents are enforced via CODEOWNERS and CI.
If the constitution changes, it is a deliberate act.


Status

Architecture locked

Footguns removed

Constitution ready to commit

Next step: minimal bootable spike


This repository is now in build mode, not design mode.
