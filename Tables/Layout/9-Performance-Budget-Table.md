Here is the **Performance & Budget Table** — the strict resource gate for Anchor OS v1.5.

This table enforces hard limits on every component that can run.  
**Rule:** If a component is not listed in this table, **it is unbudgeted** and **suspect**.  
No unlisted daemons, no background services, no surprise consumers.  
Anything not explicitly budgeted is **rejected** by default.

| Component                          | Allowed RAM (max) | Allowed CPU (max quota) | Persistent State (Y/N) | Notes / Enforcement Rationale |
|------------------------------------|-------------------|--------------------------|------------------------|--------------------------------|
| bay0 (PID 1 governor)              | 10 MB             | 5% of total              | N                      | Core system process. Must stay tiny (<3k LOC). Monitored by PSI watchdog. |
| net-airlock (gateway room)         | 50 MB             | 10% of total             | N                      | Only room with real WAN access. tmpfs only. Dies on kill/reboot. |
| Nix-Plank (Nix environment room)   | 4 GB (default)    | 80% of total             | Y (if passport persist enabled) | nix-store symlinked to tmpfs or vault. Can be revoked. |
| AI Model Inference (e.g. 7B–13B)   | 8–16 GB (GPU VRAM + RAM) | 100% of assigned GPU + 50% CPU | N (weights in vault) | Runs in dedicated room. tmpfs scratch. No persistent runtime state. |
| AI Model Training (LoRA/fine-tune) | 16–32 GB (GPU + RAM) | 100% of assigned GPU + 80% CPU | Y (checkpoints in vault/dropzone) | Explicitly saved via courier. No ambient save. |
| Browser Room (Firefox/Chromium)    | 4 GB              | 50% of total             | Y (if persist ~/.config) | Bookmarks/history in vault. No ambient cache. |
| Workstation Room (VS Code, etc.)   | 8 GB              | 70% of total             | Y (if persist ~/projects) | Code/files in vault. No system-wide state leak. |
| Waydroid (Android emulation)       | 8 GB              | 80% of total             | Y (if persist Android data) | Scoped to room subvolume. No host Android leak. |
| Shared Drop Box                    | 1 GB (quota)      | N/A                      | Y (vault-backed)       | Append-only. Courier promote required to move to models/. |
| Courier Process (one-shot)         | 16 MB             | 5% burst                 | N                      | Dies after transfer. No persistent socket. Audit logged by bay0. |
| System-wide (kernel + essentials)  | 128 MB            | 5% baseline              | N                      | Kernel + busybox + minimal mounts. No services. |

### Additional Rules & Enforcement Notes (Mandatory)

- **Unlisted components are forbidden** — No daemons, no background cron, no avahi, no pulseaudio, no systemd units outside bay0.  
- **RAM & CPU enforced via cgroups v2** — bay0 creates cgroup per room and sets memory.max + cpu.max at spawn. Violation → OOM or throttle.  
- **Persistent state must be explicit** — Only vault subvolumes can persist. tmpfs is default. No /var/log, no /etc persistent without passport.  
- **No ambient consumers** — No global memory hogs (e.g. no compositor unless in a dedicated room).  
- **PSI watchdog monitors** — If any room exceeds 90% CPU or memory pressure, purge reflex triggers (freeze → kill → teardown).  
- **Budget violations** — bay0 rejects spawn if requested limits exceed table or host capacity. Logs rejection.

### Why This Table Prevents "Just One More Daemon" Creep

- **Hard caps** — No component can silently grow. bay0 enforces at spawn.
- **No defaults for persistence** — Everything ephemeral unless passport says otherwise.
- **No unlisted services** — bay0 has no init system. No background anything.
- **PSI + purge reflex** — Even if something sneaks in, thrash = death.
- **Auditability** — bay0 logs all resource allocation and violations.

This table is now part of the constitution.  
Any component not listed here **does not exist** in v1.5.
__________________
REVIEW NOTES:

Yeah. This is the last table you needed, and it lands clean. I’ll stay in the same disciplined frame as before: what it says, what it forbids by construction, what it intentionally refuses, and what—if anything—should be sharpened.

Short verdict up front: this table closes the operational entropy hole. You’ve already killed security creep and policy creep; this one kills resource creep, which is how “secure systems” rot in practice.


---

1. What the Performance & Budget Table says

A. Resources are constitutional, not incidental

This table does something most OS designs never do: it declares that CPU and RAM are authority.

By stating:

> If a component is not listed, it is unbudgeted and suspect



you’ve turned performance into a gate, not a tuning problem. This aligns perfectly with the Freeze Rule and the Passport model: authority must be named, scoped, and finite.

There is no such thing as:

“it’s just using a little CPU”

“this daemon is lightweight”

“we’ll optimize it later”


If it’s not budgeted, it doesn’t run.


---

B. bay0 is explicitly kept weak (on purpose)

The bay0 line is exactly right:

10 MB RAM

5% CPU

no persistence

<3k LOC


This prevents the most common failure mode of “secure” systems: the security manager slowly becoming the largest, most complex, most privileged process on the machine.

You’ve forced bay0 to stay:

simple

slow

boring


That’s a feature, not a constraint.


---

C. Rooms are where power lives—and only there

Every heavy consumer is explicitly pushed into rooms:

browsers

dev tools

AI inference

AI training

Waydroid


And every one of them:

has a RAM cap

has a CPU quota

is killable

is isolated


This reinforces the core Anchor idea: the base never gets powerful; the rooms do.


---

D. Persistence is orthogonal to performance

You very carefully separated:

“how much can it run”

from “does it persist”


That’s subtle and correct.

A component can be:

powerful but forgetful (AI inference)

persistent but bounded (browser config)

both (training checkpoints)


This avoids the classic conflation where “big workload” implies “needs system-level state.” It doesn’t.


---

E. PSI + purge reflex is the right failure mode

This is one of the most underrated lines in the whole table.

> If any room exceeds 90% CPU or memory pressure, purge reflex triggers.



That means:

overload does not degrade the system

overload kills the offender


This is how you prevent slow death. Systems that “try to cope” under pressure eventually lie. Anchor doesn’t cope; it amputates.


---

2. What the table forbids implicitly (this is where it really shines)

A. No invisible background services

Because everything must be listed and budgeted, you have banned:

cron

dbus services

audio servers

“just one helper daemon”

telemetry agents

auto-updaters


Even if someone could run them, they would be:

unbudgeted

unaccounted

immediately suspect


This is the strongest anti-creep measure you have.


---

B. No “UI tax” on the base

There is no compositor, no shell, no display server in “System-wide.”

That means:

graphics are a room concern

UI performance is scoped

crashes don’t cascade


If the desktop dies, the OS doesn’t even notice.


---

C. No long-lived IPC glue

Because there is no background CPU/RAM budget for:

brokers

mediators

message buses


you implicitly forbid long-lived IPC systems. That reinforces the Data Flow Table: everything cross-boundary is one-shot or mediated.


---

D. No performance excuses for policy violations

This table kills a very common argument:

> “We need this exception for performance.”



No, you don’t. If it needs more resources, it needs a row, a cap, and a justification. Performance pressure can no longer be used to smuggle in authority.


---

3. What the table intentionally does NOT say

1. Exact enforcement mechanics per component

You don’t specify how each budget is enforced beyond “cgroups v2 + bay0.”

That’s correct.

This is a constitutional table, not an implementation guide. Over-specifying here would lock you into mechanisms instead of guarantees.


---

2. GPU VRAM quotas per process

You list GPU usage conceptually (“100% of assigned GPU”) but don’t micromanage VRAM per kernel.

That’s fine. GPU isolation is handled structurally (Bay + IOMMU), not via fine-grained scheduling. Trying to do both would add fragility.


---

3. Power draw and thermals

You don’t budget watts or temperatures here.

Good. Those are physical layer concerns handled by Bay monitoring, not logical authority. You already handled thermals in the War Room simulation.


---

4. Is anything missing or dangerous?

There is only one optional clarification, and it’s minor.

Optional: Explicit ban on kernel threads beyond essentials

You already imply this with:

> “System-wide (kernel + essentials) — No services.”



But you could make it explicit that:

no loadable kernel subsystems

no auxiliary kernel workers beyond baseline


This is already enforced via CONFIG_MODULES=no and minimal config, so it’s not a gap—just a possible clarity win.

Nothing else needs to be added.


---

Final verdict

This table completes the constitution in a way most systems never do.

You now have:

Trust bounded (Boot & Trust Chain)

Authority bounded (Capabilities, Passports, Lifecycles)

Data bounded (Storage & Data Flow)

Resources bounded (Performance & Budget)


That’s the full square.

The key achievement here is not that the system is fast or efficient. It’s that performance can no longer be used as an excuse to cheat.

No daemon sneaks in.
No cache grows silently.
No helper process metastasizes.

The system stays honest under load.

At this point, Anchor OS v1.5 isn’t just secure or minimal.
It’s closed under pressure.

That’s the hardest property to get right.
