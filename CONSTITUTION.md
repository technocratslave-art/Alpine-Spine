CONSTITUTION.md

Alpine Spine / bay0 — Constitutional Law

Status: v0.1 (Frozen)
Scope: Governing invariants for architecture, security, and operation
Audience: Maintainers, auditors, inheritors
Amendment rule: Replace-only, never mutate

PREAMBLE

This system exists to provide deterministic computing under adversarial conditions.

It assumes compromise, rejects recovery-by-repair, and enforces safety through immutability, isolation, and reset.

This Constitution defines what must never change, regardless of feature growth, tooling, or platform.

If an implementation violates this document, the implementation is wrong.


ARTICLE I — THE SPINE

§1. Definition

The Spine is the immutable base of the system.

It is not an application platform.
It is not a mutable operating system.
It is firmware.

§2. Invariants

The Spine MUST:

1. Be immutable at runtime


2. Be mounted read-only


3. Be replace-only (no in-place updates)


4. Be verified cryptographically at boot


5. Contain no user data


6. Contain no package manager


7. Contain no writable configuration



The Spine MUST NOT:

Accept runtime modification

Execute self-updates

Expose writable system paths

Depend on network availability

Host user workloads


§3. Enforcement

Immutability is enforced by:

SquashFS

dm-verity

Kernel enforcement

bay0 validation

Absence of write paths


Violation results in halt or reboot, not repair.


ARTICLE II — BAY0 (PID 1 AUTHORITY)

§1. Definition

bay0 is the sole control plane of the system.

It runs as PID 1 and is the only authority permitted to:

Launch execution units

Mount persistent storage

Create namespaces

Enforce limits

Interpret policy


§2. Exclusivity

The system MUST NOT contain:

systemd

OpenRC

Supervisord

Secondary init systems

Competing service managers


There is exactly one governor.


§3. Failure Semantics

If bay0:

Crashes

Deadlocks

Stops progressing

Fails to tickle the watchdog


Then the system MUST reboot.

There is no degraded mode.


ARTICLE III — PERSONAS

§1. Definition

A Persona is a disposable execution environment.

Personas are not trusted. Personas are not persistent. Personas are not privileged.


§2. Isolation

Every Persona MUST run in:

Its own PID namespace

Its own mount namespace

Its own network namespace

Its own IPC namespace

Its own UTS namespace


No Persona may see another Persona or the host.


§3. Limits

Every Persona MUST be bounded by:

CPU limits (cgroups)

Memory limits (cgroups)

Process/thread limits (cgroups)

Syscall limits (seccomp)


Limits are enforced by the kernel, not policy.


§4. Disposal

If a Persona is compromised:

It MUST be terminable

It MUST be replaceable

It MUST NOT require repair


Replacement is the only remediation.


ARTICLE IV — PERSISTENCE

§1. Definition

/persist is the only durable storage.

It exists solely for data, never authority.


§2. Rules

/persist MUST:

Be encrypted (LUKS)

Be mounted nodev,nosuid,noexec

Be unlocked explicitly by the user

Survive reboot


Personas MAY:

Write only to their own subtrees

Never write to /persist root

Never execute directly from /persist unless explicitly permitted


§3. Bridge Rule

> If data did not cross /persist deliberately, it does not exist.

Anything not written to /persist is ephemeral and must die on reboot.


ARTICLE V — BOOT AND RECOVERY

§1. Boot Law

Boot order is mechanical and final:

A → B → poweroff

No exceptions.


§2. Prohibitions

The system MUST NOT provide:

Recovery shells

Emergency consoles

Safe modes

Interactive boot menus

Debug entry points


Boot failure results in power off, not user intervention.


§3. Authentication

The only allowed boot-time interaction is:

LUKS passphrase entry


Failure to authenticate results in poweroff.


ARTICLE VI — UPDATES

§1. Replace-Only Rule

All updates MUST:

Replace the Spine entirely

Be written to the inactive slot

Be verified before activation

Never modify the active Spine



§2. Last Known Good

The system MUST always retain:

One active Spine

One last-known-good Spine


If a new Spine fails verification, the system reverts automatically.



ARTICLE VII — SECURITY MODEL

§1. Assumptions

The system assumes:

Zero-day exploits exist

Applications will be compromised

Networks are hostile

Users make mistakes



§2. Guarantees

The system guarantees:

No silent persistence

No lateral movement

No mutable base

No hidden authority

Deterministic reset on failure



§3. Out of Scope (v0.1)

The system does NOT defend against:

Physical attackers modifying boot firmware

Kernel memory corruption (session-only)

DMA attacks

Firmware compromise


These are explicitly deferred.


ARTICLE VIII — FAILURE AS A FEATURE

§1. Reset Doctrine

Reboot is not failure.

Reboot is resolution.

If containment is uncertain, the system MUST reset rather than guess.


§2. No Graceful Degradation

There is no partial trust state.

The system is either valid or halted.


ARTICLE IX — AMENDMENTS

§1. Amendment Rule

This Constitution MAY ONLY be amended by:

Full replacement

Explicit version bump

Maintainer consensus

Recorded rationale


§2. Forbidden Amendments

The following MAY NEVER be amended away:

Spine immutability

Single PID-1 authority

No recovery shell

Replace-only updates

Disposable personas


Any attempt to weaken these invalidates the system.


CLOSING STATEMENT

This system does not promise safety through complexity.

It promises safety through finality.

> Execution is allowed.
Persistence is not.
Authority is singular.
Power ends all arguments.
