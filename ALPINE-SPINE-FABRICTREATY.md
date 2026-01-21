ALPINE-SPINE-FABRIC-TREATY.md

Status: Constitutional Extension (v0.1)
Scope: Host ↔ Persona Boundary Enforcement
Applies to: Alpine Spine / bay0 systems only


PURPOSE

The Fabric Treaty defines the non-negotiable separation of power between the immutable host (Spine / bay0) and all execution environments (Personas).

This treaty exists to prevent:

State bleed

Authority inversion

Long-lived compromise

“Helpful” abstractions that weaken isolation


The Fabric Treaty is enforced by kernel mechanics, not conventions.


DEFINITIONS

Spine
The immutable host system. Firmware-grade. Replace-only.

bay0
PID 1 governor. Sole control plane. Owns lifecycle, power, and boundaries.

Persona
A disposable execution island. Signed. Bounded. Expendable.

Fabric
The sum of kernel primitives enforcing separation:

Namespaces

cgroups

seccomp

mount topology

device filtering

power reset


CORE LAW

LAW 1 — SINGLE SOVEREIGN

There is exactly one authority on the system: bay0.

bay0 is PID 1

bay0 launches everything

bay0 terminates everything

bay0 owns reboot and poweroff

bay0 cannot be replaced at runtime


No secondary init systems.
No nested supervisors.
No competing control planes.

Violation = halt.

LAW 2 — HOST IS NOT A WORKSPACE

The Spine is not a place where work happens.

No shells for users

No package managers

No writable root

No services beyond bay0 and the watchdog

No “debug just this once”

If work requires persistence, networking, UI, or tools, it belongs in a Persona.


LAW 3 — PERSONAS ARE EXPOSED BY DESIGN

Personas are allowed to be compromised.

This is intentional.

Security is achieved by:

Short lifetimes

Hard ceilings

Zero trust between personas

Deterministic purge

Reboot as reset

A Persona may fall.
The Spine must not.


LAW 4 — NO SHARED STATE WITHOUT CEREMONY

There is no ambient sharing.

Forbidden:

Shared writable mounts

Shared IPC

Shared sockets

Shared memory

Implicit clipboard

Background sync


Allowed:

Explicit transfer via Umbilical API

User-initiated, logged, pull-based actions

Staging through /persist/transfer


If data crosses personas, it must cross deliberately.


LAW 5 — PERSISTENCE IS A PRIVILEGE

Persistence exists only in /persist, and only by contract.

Rules:

/persist mounted nodev,nosuid,noexec

Personas see only their own subtrees

Exec requires explicit bind-remount

No persona can modify inventory, passports, or Spine artifacts


If it didn’t cross /persist intentionally, it doesn’t survive.



LAW 6 — POWER ENDS ALL ARGUMENTS

Recovery is not negotiation.

Boot sequence is mechanical:

Slot A → Slot B → Poweroff

No recovery shell

No menu

No safe mode


Runtime containment failure results in:

Persona termination, or

Host reboot
Power reset is the final authority.


ZERO-DAY POSTURE (HONEST)

What the Treaty Guarantees

Remote compromise is contained to a Persona

Lateral movement is blocked by construction

Persistence requires explicit user action

Reboot clears all volatile compromise

Spine state cannot be modified at runtime


What the Treaty Does NOT Pretend

Kernel 0-days can escape containment

Physical attackers can replace the ESP

Memory-resident attacks exist until reboot


Mitigation strategy:
Short sessions + immutability + reset
—not detection fantasies.


ENFORCEMENT MECHANISMS

The Fabric Treaty is enforced by:

Namespaces: PID, mount, net, IPC, UTS

cgroups v2: CPU, memory, pids

seccomp: syscall denial (no namespaces, no ptrace, no kexec)

Mount geometry: read-only Spine, noexec persist

Minimal /dev: tmpfs + explicit mknod

Watchdog: bay0 liveness enforced by hardware

Signatures: signify-verified artifacts only


No policy engine may override these.


NON-GOALS (EXPLICIT)

The Fabric Treaty does not attempt to:

Make exploitation impossible

Detect clever attackers

Preserve compromised state

Heal a broken host

Protect against physical possession

Those goals create fragile systems.
This system chooses determinism instead.


EXTENSION RULES

The Fabric Treaty may only be extended if:

1. The Spine remains immutable


2. bay0 remains sole authority


3. Personas remain disposable


4. Reboot remains absolute


5. No new shared mutable state is introduced


Any feature violating these is rejected by default.


FINAL STATEMENT

This system is not secure because it is clever.
It is secure because it is short, boring, and honest.

The Fabric Treaty ensures that:

Power flows one way

State has clear borders

Failure is survivable

Recovery is mechanical


Everything else is expendable.


End of Treaty.
No soft interpretations permitted.

>
Yes — exactly. Cleanly and mechanically.

bay0 pins cores, caps CPU, and owns scheduling authority for everything that runs above it.

Here’s the precise shape of it, without fluff.

bay0 is not a “policy engine.” It is a resource allocator with veto power.

What bay0 directly controls:

• CPU core pinning
Personas are placed into cpusets.
Example:

web persona → cores 2–3

work persona → cores 4–7

vault persona → core 1

bay0 → core 0 only


Personas never see or migrate onto bay0’s core.

• CPU time (hard ceilings)
Using cgroup v2 cpu.max:

burst allowed

sustained usage capped

starvation impossible for bay0


If a persona misbehaves, it slows — it does not destabilize the host.

• Memory ceilings
memory.max is enforced per persona.

No overcommit to host

No swap abuse

OOM kills the persona, not the system


• Thread / fork limits
pids.max enforced.

Fork bombs die locally

bay0 remains responsive


• I/O pressure awareness (optional, later)
PSI can be read by bay0 to:

preemptively kill a persona

or trigger a reboot if containment degrades


What bay0 explicitly does not do:

• It does not “balance workloads”
• It does not autoscale
• It does not optimize throughput
• It does not chase efficiency

That’s intentional.

This is containment-first scheduling, not performance-first scheduling.

Why this matters for zero-day posture:

If a persona is hit:

It can spike CPU → capped

It can fork → capped

It can allocate memory → capped

It can hang → watchdog resets host

It cannot starve bay0

It cannot persist without /persist access

It cannot migrate into host scheduling space


If the host is hit (kernel 0-day):

Attacker lives only until power loss

Cannot persist into Spine

Cannot survive reboot

Cannot poison future boots (dm-verity)


So the model is:

bay0 = hard physical governor
personas = burnable workloads

That’s why the Fabric Treaty still holds.
That’s why Android Studio “failing” is expected unless it’s placed in a high-ceiling dev persona.
That’s why docks/modules are fine later — at their own risk tier.

