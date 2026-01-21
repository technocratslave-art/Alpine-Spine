SECURITY.md

Alpine Spine / bay0 — Security Model

Status: v0.1 (Constitutionally Frozen)
Scope: Runtime security, containment, persistence denial, recovery semantics
Audience: Security auditors, senior engineers, maintainers

1. Security Philosophy (Read This First)

This system does not attempt to prevent compromise at all costs.
It assumes compromise and is engineered so that compromise:
cannot persist,
cannot spread laterally,
cannot silently escalate,
and cannot survive reboot.


> Security goal:
Deny persistence and authority, not bugs.


Zero-day exploits are tolerated.
Persistence without consent is structurally denied.


2. Trust Boundaries

2.1 Root of Trust (v0.1)

Trusted components:

1. Offline build machine


2. Ed25519 signing key (signify)


3. Physical custody of the device



Untrusted by default:

Network

Applications

Personas

Containers

User actions

Runtime state

Memory contents

3. System Layers and Authority

3.1 Spine (Immutable Host)

Alpine Linux minimal base

Packaged as SquashFS

Mounted read-only

Verified with dm-verity

No package manager

No writable paths

No mutable configuration


Authority: absolute
Mutability: none at runtime
Persistence: none

> The Spine is firmware, not an OS.


3.2 bay0 (PID-1 Governor)

Runs as PID 1

Single control plane

No systemd / OpenRC

No secondary supervisors

Enforces all security invariants


Responsibilities:

Platform integrity validation
Persona lifecycle control
Namespace creation
cgroup enforcement
seccomp enforcement
Watchdog management
Inventory verification


If bay0 fails, stalls, or wedges → kernel watchdog reboots system.


3.3 Personas (Execution Units)
Signed SquashFS images

Strictly isolated via:
PID namespaces
Mount namespaces
Network namespaces
IPC / UTS namespaces

Resource-bounded with cgroups v2

Syscall-restricted with seccomp

Minimal /dev (tmpfs + explicit mknod)


Authority: none
Persistence: only via explicit /persist bind mounts
Disposability: guaranteed


3.4 /persist (Data Only)
LUKS2 encrypted ext4
Mounted with: nodev,nosuid,noexec
Survives reboot
Contains data only, never authority

Only bay0 may write to /persist root.
Personas see only their own subtrees.

4. Boot and Reset Security

4.1 Boot State Machine

Power On
 → Bootloader
 → Kernel
 → initramfs
 → dm-verity verification
     A → B → poweroff
 → LUKS unlock (/persist)
     3 attempts → poweroff
 → switch_root
 → bay0 (PID 1)
 → invariants checked
 → watchdog armed
 → personas launched

4.2 No Recovery Paths

Explicitly not present:

No recovery shell

No emergency mode

No GRUB rescue

No interactive boot menu

No on-device reinitialization


Recovery is physical (offline reflash only).


5. Update Security

A/B slots (SPINE_A, SPINE_B)
Replace-only updates
No in-place mutation

New Spine must:
verify via dm-verity
survive 5 minutes uptime
pass bay0 invariant checks

Only then marked “good”

Rollback is automatic on verification failure.


6. Zero-Day Threat Model

6.1 What This System Defends Against

✅ Remote code execution
✅ Browser exploits
✅ Malicious containers
✅ Supply-chain attacks inside Personas
✅ Lateral movement between workloads
✅ Persistence across reboot
✅ Silent privilege escalation
✅ Configuration drift


6.2 What This System Does Not Defend Against (v0.1)

❌ Physical attacker modifying boot partition (ESP)
❌ Kernel memory corruption (session-only)
❌ DMA attacks
❌ Firmware compromise
❌ User losing LUKS passphrase

These are explicitly out of scope for v0.1 and documented honestly.

Mitigation is planned for v0.2 (Secure Boot + TPM).


7. Zero-Day Handling Strategy

Attack Location	Result

Persona	Discard persona
Kernel	Reboot clears attack
bay0	Watchdog reboots
Memory	Lost on power cycle
Disk (Spine)	Immutable
Disk (/persist)	Data only, no authority


> If it can’t persist, it can’t own.


8. Watchdog Enforcement

Kernel watchdog (/dev/watchdog)

Opened by bay0 at init

Ticked from main loop

Never closed


Failure modes:

bay0 deadlock → reboot

bay0 crash → reboot

infinite loop → reboot

No userspace process can suppress this.


9. Containment Guarantees

9.1 Namespaces

Personas cannot see:

host processes

other personas

host mounts

host network


PID namespace uses double-fork to ensure persona init is PID 1


9.2 cgroups v2

CPU quotas enforced

Memory hard limits enforced

PID limits enforced

Fork bombs fail deterministically


9.3 seccomp

Blocked syscalls include (non-exhaustive):

ptrace

mount, pivot_root

kexec*

bpf

perf_event_open

namespace creation syscalls


Threads are allowed; namespace creation is not.


10. Device and Network Security

10.1 /dev

tmpfs only

Explicit device nodes via mknod

No block devices

No raw memory

GPU devices only if passport allows


10.2 Network

Per-persona network namespace

veth pairs

nftables with default-drop

RFC1918 filtering enforced

No CAP_NET_ADMIN in personas


11. Persistence Rules

What survives reboot:

Files deliberately written to /persist

Signed persona inventory

Boot selector


What never survives reboot:

Running code

Memory implants

Exploit payloads

Unauthorized changes

> Reboot is absolution.


12. Reporting Security Issues

This project does not run a public bug bounty.

If you find:

a persistence bypass

a containment failure

an authority escalation

a violation of documented invariants


Please report privately to the maintainers listed in CODEOWNERS.

Include:

exact reproduction steps

kernel version

persona passport used

whether persistence survived reboot


13. Security Guarantees (Final)

This system guarantees:

No silent persistence

No shared authority

No recovery shell escalation

No mutable base

No drift without replacement

Deterministic reset on failure


It does not guarantee:

No bugs

No exploits

No downtime

No data loss if user errs


14. One-Line Security Proof

> An attacker may execute, but cannot endure.


End of SECURITY.md
