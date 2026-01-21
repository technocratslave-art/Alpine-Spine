Bay0 is the control plane. It is not “an OS.” It is PID 1, an event loop, and a set of ceremonies that create/kill isolated execution islands (personas) and enforce the Spine invariants. Below is the full role map: what Bay0 is responsible for, what it must never do, everything it touches, and how the pieces interact.

Bay0’s job in one sentence: own the hardware-facing truth, enforce the constitution, and make all workloads disposable.


Bay0 roles (authoritative list, v0.1)

1. PID-1 reaper and liveness governor
Bay0 is PID 1, so it must:


Reap zombies deterministically (SIGCHLD loop, waitpid)
Never block indefinitely in a way that prevents watchdog tickle
Own the main loop cadence (tick watchdog, reap, dispatch events)


What it touches:

waitpid() / SIGCHLD

/proc (minimal reads only if needed for child tracking)

internal “persona table” state


2. Platform integrity gate at boot
Before Bay0 allows any persona, it verifies invariants. If any fail: halt/reboot.


Checks Bay0 enforces (typical v0.1 set):
Root filesystem is read-only SquashFS
dm-verity device is active for the Spine root
No runtime package manager present
No setuid/setgid binaries
No file capabilities

/persist mount flags correct: nodev,nosuid,noexec

Kernel lockdown toggles set (module loading disabled if you’re using that invariant)
Inventory and persona signatures verify


What it touches:

/proc/mounts or /proc/self/mountinfo

/sys (dm-verity status, watchdog, cgroups)

/etc/spine/master.pub

the signed inventory artifact on /persist


3. Watchdog authority (power ends arguments)
Bay0 opens /dev/watchdog early, keeps the fd open, and tickles it on schedule. If Bay0 hangs/crashes: hardware reboots.



What it touches:

/dev/watchdog

/sys/class/watchdog/watchdog0/* (optional reads)

timer/sleep cadence in the main loop


What it must never do:

Close watchdog fd (closing can trigger reboot on many drivers)

Allow “other services” to become the system liveness owner


4. /persist gatekeeper (the only durable bridge)
Bay0 is the only writer to /persist root (except allowed subtrees). It mounts /persist with nodev,nosuid,noexec, controls bind mounts into personas, and enforces “persist_exec” policy via bind remount.



What it touches:

/dev/mapper/<persist> (LUKS mapping created by initramfs or bay0—your design)

/persist/**

mount syscalls: mount(MS_BIND), mount(MS_REMOUNT...)


What it must never do:

Mount /persist exec globally

Give personas access to /persist root

Let persona-created mounts propagate back to host (must set MS_PRIVATE)


5. Persona lifecycle manager (create → launch → monitor → purge)
This is the heart. Bay0 owns the only blessed way to start/stop workloads.



Lifecycle phases Bay0 runs:

Verify persona image signature

Parse/validate passport schema (deny unknown fields)

Create cgroup v2 subtree + apply ceilings (cpu/memory/pids)

Create namespace boundary (mount/pid/net/ipc/uts)

Construct persona filesystem view (mount lower, tmp upper, bind home, etc.)

Create minimal /dev (tmpfs + explicit nodes)

Configure network (if allowed) via host-side veth + nftables

Apply seccomp filter (thread-safe policy)

Exec persona init as PID 1 in new PID namespace (double-fork pattern)

Track persona state + enforce termination policy


What it touches:

/persist/personas/*.sqsh (+ sig)

/sys/fs/cgroup/* (create dirs, write controls)

unshare(), fork(), execve(), setns() (if used)

mount() operations under /persona/<id>/...

minimal device nodes under /persona/<id>/dev

internal state DB/log


What it must never do:

Start workloads outside this ceremony

Allow “manual” privileged exec into host

Allow nested control planes inside personas (systemd, init managers that violate rules—unless you explicitly allow in that persona type)


6. Network policy authority (host-side only)
Bay0 is the only code allowed to create host networking primitives for personas.



For networked personas, Bay0:

Creates veth pair in host netns

Moves guest end into persona netns

Attaches host end to a bridge (or routed setup)

Applies nftables policies (deny-by-default) per persona/mode

(Optionally) configures NAT/forward rules if WAN access is allowed


What it touches:

ip link ..., ip netns ... (or netlink directly)

nftables: nft -f /run/persona/<id>/rules.nft

bridge interface br-personas (or equivalent)

/run/persona/<id>/ ephemeral rules staging


What it must never do:

Flush global tables shared with other system networking (avoid flush table inet ... globally)

Permit persona to author host firewall state

Allow Bay0 to become a “networked daemon” itself (control plane should be network-denied)


7. Device exposure broker (/dev is a contract)
Bay0 decides what hardware a persona can see. The default is “almost nothing.”



Mechanism:

Persona gets tmpfs /dev

Bay0 creates only safe char devices (null, zero, urandom, tty, ptmx, etc.)

Optional device pass-through is explicit and audited (GPU, audio, etc.) via bind-mounting specific device nodes and enforcing cgroup device allowlist (if you use it)


What it touches:

mknod(), tmpfs mount

/dev/dri/* (if GPU allowed)

cgroup device control (if enabled)


What it must never do:

Mount devtmpfs and “delete what you don’t want” (racy, leaky)

Expose raw block devices to untrusted personas


8. Umbilical gate (the only cross-persona corridor)
If personas can exchange anything, it must be through Bay0’s narrow, logged API.



Bay0 provides:

Local Unix socket endpoint (host-side)

Peer credential authentication (SO_PEERCRED)

Explicit operations (transfer request, approve, copy-out/copy-in, start/stop)

Staging area under /persist/transfer/ (or /run depending on durability choice)

No ambient sharing; pull-based approval


What it touches:

/var/run/bay0/umbilical.sock (or /run/...)

/persist/transfer/... staging

event log entries tied to persona IDs (no sensitive payload logging)


What it must never do:

Provide a “shell” escape hatch

Provide arbitrary command execution APIs


9. Audit and minimal logging (evidence without secrets)
Bay0 records events necessary for security review and debugging, without leaking content.



Typical audit log events:

Boot slot selected / fallback occurred

Integrity check pass/fail

Persona start/stop/purge

Signature verification results

Network mode applied

Umbilical transfers (metadata only)


What it touches:

/persist/logs/bay0.log (if durable logs are allowed) or /run/log/... (if volatile)

state DB (SQLite or simple append log)


What it must never do:

Log secrets, file contents, clipboard contents, passwords

Become a general syslog sink


10. Fail-closed decision maker
When Bay0 detects an invariant violation or containment failure, it does not “try harder.” It stops.



Fail-closed actions:

Halt boot (don’t start personas)

Purge persona (fast kill)

Reboot if containment fails or Bay0 is compromised/suspected hung

Poweroff in initramfs when both slots fail / LUKS fails


What it touches:

reboot(LINUX_REBOOT_CMD_RESTART) (or writes to sysrq trigger if you use that)

watchdog (indirect reboot if Bay0 stops ticking)


Everything Bay0 touches (inventory list)

Host filesystem / mounts:

/ (read-only root)

/etc/spine/master.pub

/run and /var/run (ephemeral runtime state)

/sys/fs/cgroup (write)

/proc (read)

/persist (write root + bind sources)

/persona/<id>/... (mount targets in host mount namespace)


Block/crypto:

/dev/mapper/persist (if Bay0 manages mapping; otherwise initramfs does)

reads/writes to raw spine slots are not Bay0 in v0.1 unless you explicitly put update writing in Bay0; recommended: Bay0 verifies + orchestrates, an offline tool writes.


Kernel interfaces:

namespaces: unshare(), fork(), execve()

mount API: mount(), propagation flags

seccomp API (via library)

watchdog char dev

nftables/netlink (host netns)


Persona artifacts:

/persist/personas/<id>.sqsh

/persist/personas/<id>.sqsh.sig

/persist/personas/inventory.toml

/persist/personas/inventory.toml.sig

How it all interacts (sequence diagrams in words)

Boot to safe runtime:

1. initramfs enforces A→B→poweroff and unlocks /persist (or Bay0 does unlock immediately after switch_root—either way, the rules are the same)
2. switch_root into verified Spine
3. Bay0 starts as PID 1
4. Bay0 runs integrity gate; if any check fails, it halts/reboots
5. Bay0 opens watchdog and begins tick loop
6. Bay0 loads and verifies inventory
7. Bay0 launches the default shell persona set (per embodiment preset) or waits for local user command through the spine CLI



Persona launch (the real ceremony):

1. Verify persona image signature
2. Read passport; deny unknown fields; enforce defaults
3. Prepare cgroup subtree and limits
4. Create namespace boundary:

fork → child unshare(PID/mount/net/ipc/uts) → fork → grandchild is PID 1

5. Host-side: create veth + attach + nftables (if network not “none”)
6. Child-side: configure lo and eth0 inside netns
7. Mount persona FS (lower = squashfs, upper = tmpfs if needed)
8. Bind mount allowed /persist/home/<id> into persona
if persist_exec allowed, remount bind target with exec (only that target)

9. Build minimal /dev tmpfs and optional device binds (GPU etc.)
10. Apply seccomp filter
11. Exec persona init (grandchild PID 1)
12. Bay0 tracks and reaps; purge on policy triggers


Containment failure rule:

If Bay0 cannot kill all processes in persona deterministically (within the containment window), Bay0 triggers reboot. That’s the “clean power to work” guarantee.


Answering your two implied security questions plainly

“I thought we had zero-days done with the security.”
You have “zero-day persistence” handled and “zero-day blast radius” bounded. You do not have “zero-day execution impossible” because that’s not realistic without hardware/boot guarantees and without assuming the kernel is perfect.

“So if it hits the host, reboot clean?”
Yes: if the host (Bay0/Spine runtime) is compromised in memory, your recovery primitive is reboot (watchdog-backed). That clears volatile compromise. What matters is preventing silent modification of:

Spine on disk (dm-verity blocks)

persona artifacts (signatures block)

persist policy (Bay0 gate + mount flags)


v0.1 still admits that a host compromise can tamper with /persist user data (documents) during that session. It cannot make the Spine silently persist altered executables without failing verity/signature gates on next boot, assuming the attacker cannot replace the boot verifier (EFI physical attack).
