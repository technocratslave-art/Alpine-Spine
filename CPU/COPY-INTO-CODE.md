Here’s an exact, “copy into code” CPU layout. two canonical mappings: 8-core (most dev laptops / x86 mini PCs) and big.LITTLE (typical phone). These are best-practice defaults for  model: bay0 never competes, work stays smooth, web stays disposable, dev gets horsepower but bounded.

8-core (0–7) baseline

Core roles

Core 0: bay0 only (control plane)

Core 1: vault / maintenance persona (offline, low noise)

Cores 2–3: web persona (untrusted, bursty)

Cores 4–7: work persona (primary human productivity)

Dev persona: uses 2–7 but throttled by cpu.max (so it can’t starve work)


cpuset assignments (cgroup v2)

bay0:

cpuset.cpus = 0

cpuset.mems = 0


vault:

cpuset.cpus = 1

cpuset.mems = 0


web:

cpuset.cpus = 2-3

cpuset.mems = 0


work:

cpuset.cpus = 4-7

cpuset.mems = 0


dev:

cpuset.cpus = 2-7

cpuset.mems = 0



cpu.max (hard ceilings) Use period 100000 (100ms). Quota is microseconds per period.

bay0: no cap (don’t set cpu.max, or set max)

cpu.max = max 100000


vault: 10% of 1 core

cpu.max = 10000 100000


web: 60% of 2 cores (i.e., 1.2 CPU total)

cpu.max = 120000 100000


work: 300% of 4 cores (i.e., 3.0 CPU total, leaves headroom)

cpu.max = 300000 100000


dev: 250% of 6 cores (i.e., 2.5 CPU total, strong but can’t bulldoze)

cpu.max = 250000 100000



Optional but recommended: cpu.weight (relative priority within contention)

bay0: 10000 (max)

work: 1000–2000

dev: 500–1000

web: 100–200

vault: 50–100


Reasoning (why these numbers)

bay0 is isolated by cpuset and also “wins” if anything weird happens.

work gets the cleanest cores and the most weight.

dev is allowed wide affinity but capped so it can’t ruin UX.

web is boxed into 2 cores and capped so browser insanity can’t take the machine hostage.


Phone big.LITTLE (example: 8 cores = 4 little + 4 big) Assume:

little cores: 0–3

big cores: 4–7


Core roles

Core 0 (little): bay0 only

Core 1 (little): vault / background

Cores 2–3 (little): web (untrusted)

Cores 4–7 (big): work and dev (but dev bounded)


cpuset

bay0: 0

vault: 1

web: 2-3

work: 4-7

dev: 2-7 (can spill onto big cores, but cpu.max prevents takeover)


cpu.max tuned for thermals (phones must be stricter)

vault: 5% of 1 core

5000 100000


web: 50% of 2 little cores (1.0 CPU total)

100000 100000


work: 200% of 4 big cores (2.0 CPU total) for sustained sanity

200000 100000


dev: 220% of 6 cores (2.2 CPU total) but still bounded

220000 100000



If you want “Android Studio / Gradle build” to feel less painful on phone, you don’t raise work; you raise dev quota temporarily via an explicit “Workshop Mode” passport, then return to normal.

Exact files and paths (so it’s implementation-ready)

Assuming cgroup v2 mounted at /sys/fs/cgroup and a tree like:

/sys/fs/cgroup/bay0.slice

/sys/fs/cgroup/persona-web.slice

/sys/fs/cgroup/persona-work.slice

/sys/fs/cgroup/persona-dev.slice

/sys/fs/cgroup/persona-vault.slice


Write these:

cpuset setup (must enable controller at parent)

In root cgroup:

echo +cpuset +cpu > /sys/fs/cgroup/cgroup.subtree_control



Then per cgroup:

echo 0 > bay0.slice/cpuset.cpus

echo 0 > bay0.slice/cpuset.mems

echo 2-3 > persona-web.slice/cpuset.cpus

echo 0 > persona-web.slice/cpuset.mems etc.


cpu.max:

echo "120000 100000" > persona-web.slice/cpu.max

echo "300000 100000" > persona-work.slice/cpu.max

echo "250000 100000" > persona-dev.slice/cpu.max

echo "10000 100000"  > persona-vault.slice/cpu.max


cpu.weight:

echo 10000 > bay0.slice/cpu.weight

echo 1500  > persona-work.slice/cpu.weight

echo 800   > persona-dev.slice/cpu.weight

echo 150   > persona-web.slice/cpu.weight

echo 80    > persona-vault.slice/cpu.weight


If you tell me which target you’re tuning first (8-core x86 box, 6-core laptop, or a specific phone SoC model), I’ll collapse this to one definitive “shipping preset” with exact core maps and quotas for that exact topology.
