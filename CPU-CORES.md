**Here’s the “no more cores” locklist — written so there’s no wiggle room where someone later says “ah, but now I can just…” and quietly turns the box into a general-purpose OS again.

This is intentionally blunt. If any item below becomes “allowed,” it must be a new schema version + new signed inventory + new constitution change. No silent exceptions.

CPU and core allocation invariants (hard “NO” list)

1. No persona may ever expand its cpuset beyond what bay0 assigns at launch time.
2. No persona may ever write to any cgroup control file (cpu., cpuset., pids., memory., io.*, any controller).
3. No persona may ever gain access to the cgroup filesystem for writing (including via bind mounts, procfs tricks, or privileged containers).
4. No persona may ever call sched_setaffinity() successfully for threads outside the persona’s allowed cpuset.
5. No persona may ever call sched_setaffinity() successfully to “tighten” itself onto bay0 cores (even if it stays “inside” cpuset by accident).
6. No persona may ever set CPU affinity on bay0 (PID 1) or any host process.
7. No persona may ever set realtime scheduling (SCHED_FIFO / SCHED_RR / SCHED_DEADLINE).
8. No persona may ever raise nice priority above its class default (i.e., cannot become “more important” than work).
9. No persona may ever change cgroup cpu.weight or cpu.max.
10. No persona may ever disable or alter PSI monitoring thresholds.
11. No persona may ever change the tick rate, scheduler knobs, or sysctls affecting CPU arbitration.
12. No persona may ever load kernel modules (including “just for perf,” “just for virtualization,” or “just for sensors”).
13. No persona may ever use perf_event_open() (profiling is a host-only activity, if allowed at all).
14. No persona may ever use eBPF (no bpf() syscall) for any reason.
15. No persona may ever mount anything (no mount/umount/pivot_root), including “just mounting cgroup” or “just mounting debugfs.”
16. No persona may ever unshare() any namespace (even inside itself) to create hidden scheduling sandboxes.
17. No persona may ever create nested containers with elevated CPU privileges (no privileged pods, no hostPID, no hostCgroup).
18. No persona may ever run a container runtime that can alter host cgroups (Docker-in-Docker style paths are forbidden).
19. No persona may ever access /sys/kernel, /sys/fs/cgroup (write), /proc/sys (write), or /sys/fs/bpf.
20. No persona may ever access /dev/cpu/* or MSR interfaces or anything that can tune frequency/boost.

Core reservation invariants (bay0 stays sovereign
21. bay0 cores are never shared with any persona, ever. Not “mostly,” not “unless idle,” not “for burst.” Never.
22. bay0’s cpuset is fixed and minimal; it cannot be “temporarily expanded” to help dev builds.
23. bay0 must not run “helpful background services” that compete for CPU (no package updates, no indexers, no telemetry, no “smart tuning daemons”).
24. bay0 must not start workloads that consume work cores “just to keep the system warm.”
25. bay0 is allowed to reboot the host if CPU containment is violated or cannot be enforced deterministically.

Work persona sovereignty invariants (your “clean power to work” promise)
26. Work persona cores are pinned and not loaned out to web by policy.
27. Web persona is never allowed to run on work-only cores, even if work is idle.
28. Dev persona is never allowed to take exclusive ownership of work-only cores.
29. If dev needs more performance, the answer is not “give it more cores,” it’s “bounded quota change in a separately signed Workshop passport,” and only if you explicitly choose that class.
30. Work persona always has higher cpu.weight than dev and web.
31. Work persona must remain usable under worst-case dev activity (bounded by cpu.max + weight).

Dev persona containment invariants (“you don’t get to become root of the machine”)
32. Dev persona cannot request “all cores” by configuration.
33. Dev persona cannot self-promote to a “dock” mode from inside the persona.
34. Dev persona cannot change its own ceilings at runtime.
35. Dev persona cannot run host-level profilers, tracers, or kernel debuggers.
36. Dev persona cannot enable KVM/virtualization if that grants broader scheduling control than the passport allows.
37. Dev persona cannot create RT threads to bypass quota enforcement.
38. Dev persona cannot disable OOM behavior, PSI actions, or watchdog policy.
39. Dev persona cannot write anything that affects global scheduler state.
40. Dev persona cannot attach to, introspect, or control other personas’ processes.

Web persona hard box invariants (untrusted stays small forever)
41. Web persona cores are strictly limited and never expanded.
42. Web persona quota is capped and never raised “because browsing feels slow.” If browsing feels slow, you add a different persona or fix the browser; you do not widen the box.
43. Web persona cannot gain GPU, camera, mic, or extra CPU privileges without a new signed passport and inventory update.
44. Web persona cannot call any interface that enumerates host topology in a way that enables smarter CPU exploitation (as much as practical: restrict perf, kmsg, privileged /proc).
45. Web persona cannot run compilers, build systems, or long-running “dev-like” workloads by policy (enforce via ceilings and/or package set).

Vault/offline persona invariants (quiet room)
46. Vault persona is CPU-low, network-none, and never promoted.
47. Vault persona is never allowed to run on bay0 cores.
48. Vault persona is never allowed to run heavy workloads “because it’s offline anyway.”
“No new cores” means no backdoors through tooling

49. No “auto-tuner” daemon is permitted to rewrite cpu.max, cpu.weight, or cpuset at runtime.
50. No “performance helper” is permitted to reassign affinities dynamically.
51. No “optimizer” is permitted to move tasks across personas to “balance load.”
52. No “scheduler AI” lives in bay0. bay0 enforces contracts; it does not invent new ones.
53. No user-facing knob in a UI may map to “more cores” unless it literally swaps to a different signed passport class (and that swap is logged, explicit, and reversible only by another explicit signed choice).
54. No persona may access host power-management controls (cpufreq governors, intel_pstate knobs, energy_perf_bias, etc.).
55. No persona may access thermal trip or cooling device controls.

Enforcement must be mechanical (not “policy”)
56. cpuset must be enforced via cgroup v2 cpuset controller, not “best effort.”
57. CPU caps must be enforced via cgroup v2 cpu.max, not “nice levels.”
58. Relative priority must be enforced via cpu.weight, not “please behave.”
59. A persona that attempts forbidden syscalls (unshare/mount/bpf/perf/ptrace) must fail deterministically (EPERM) every time.
60. Any detected mismatch between intended cpuset/quota and actual kernel state is a fatal integrity failure: bay0 halts or reboots (your choice, but it must be final and automatic).
61. If a persona cannot be contained (D-state tasks, stuck IO) within the purge window, host reboots. No negotiation.

Audit and “no drift” requirements (so nobody quietly loosens it)
62. The passport must declare: cpuset class, cpu.max, cpu.weight, and pids.max (threads). If it’s missing, bay0 refuses to launch.
63. Inventory must bind persona ID → exact passport schema version. Mismatch = refuse.
64. bay0 must log: persona ID, cpuset, cpu.max, cpu.weight, and the measured kernel values after applying them.
65. CI must fail if a change touches default CPU maps, weights, or quotas without updating the schema version and the constitutional docs.
66. No runtime “manual override” command exists that changes cores/quotas without going through the signed update ceremony.
67. No “debug build” bypass exists in release artifacts.



One-line constitutional rule that captures all of this

68. “No persona may ever obtain more CPU cores, more CPU time, or higher CPU priority than its signed passport grants — and any attempt or mismatch is treated as containment failure.”

