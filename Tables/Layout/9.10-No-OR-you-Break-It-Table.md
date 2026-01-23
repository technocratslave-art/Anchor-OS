Here is the **No or You Break it** **Non-Goals Table** for Anchor OS v1.5 — the definitive “negative space” document.

This table lists everything Anchor explicitly **does not** support in v1.5.  
It prevents reopening settled debates, bikeshedding, scope creep, and “just this once” exceptions.  
If something is listed here, it is **off the table** until at least v2.0 (and even then, only after full constitutional review).

| Explicitly Not Supported in v1.5                          | Reason (Why It Is a Non-Goal)                                                                                          | “Consider in v2?” (Yes/No) | Additional Notes / Rationale |
|-----------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|----------------------------|------------------------------|
| Ambient / global clipboard                                | Creates silent cross-room channel → covert exfil & tracking risk. Violates #7 Ambient IPC invariant.                  | No                         | Session clipboard is already allowed (scoped, visible, revocable). Global is unnecessary and dangerous. |
| Runtime policy change in running room                     | Enables privilege escalation → room can gain capabilities after spawn. Violates #11 Capability Escalation invariant. | No                         | Passports are spawn-time only. Runtime change = policy rot. |
| Dynamic/scripted/conditional passports                    | Turns passport into a policy language → un-auditable, complex, rot-prone. Violates #5 flat passport freeze rule.      | No                         | Flat TOML is the line. Expressive policy = slow death. |
| Persistent courier / reusable connection                  | Creates long-lived channel → persistent exfil risk. Violates #6 Cross-Room Data Flow & #4 bounded lifetime.           | No                         | Courier is one-shot by design. Reusable = daemon. |
| Auto-persistence of room state (default on)               | Creates silent persistence → malware survives reboot. Violates #5 Room Persistence & “forgetful by default”.          | No                         | Persistence is explicit (passport + prompt). Default on = betrayal of core philosophy. |
| Arbitrary device access (wildcards, no passport)          | Enables DMA/privacy leaks → room can access unmapped hardware. Violates #13 Device Access.                             | No                         | Bind-mount only allowed devices. Wildcards = surprise access. |
| Base runtime flexibility (live edits, modules)            | Enables drift & attack surface growth in trusted base. Violates #1 Base & #16 Runtime Flexibility.                    | No                         | Base is immutable firmware. Runtime flexibility belongs only in rooms. |
| Global shared memory / IPC                                | Creates ambient cross-room channel → tracking & exfil. Violates #7 Ambient IPC & #15 Ambient Authority.               | No                         | No shared /dev/shm, no persistent sockets. Explicit courier only. |
| Invisible / hidden capabilities                           | Creates surprise state → users don’t know what’s active. Violates #2 Visible On/Off State freeze rule.                | No                         | All capabilities require visible indicator. No stealth mode. |
| Background daemons / services in base                     | Creates persistent processes → attack surface & drift. Violates #18 Auditability & #4 PID 1 Authority.               | No                         | bay0 is the only PID 1. No systemd, no cron, no anything. |
| Auto-restart of killed rooms                              | Creates unbounded lifetime → zombie processes & persistence. Violates #4 Bounded Lifetime freeze rule.               | No                         | Room death = permanent until explicit respawn. No auto-revival. |
| Global / system-wide network access                       | Creates ambient exfil → tracking & lateral movement. Violates #12 Network Access & #15 Ambient Authority.             | No                         | Network is room-scoped. net-airlock is the only WAN path. |
| Unmediated hardware passthrough (no IOMMU)                | Enables DMA attacks → host memory compromise. Violates #3 DMA Breakout fix & Invariant 0 (Secure Measured Boot).      | No                         | IOMMU mandatory at boot. Panic if disabled. |
| Recovery shell / debug mode on boot failure               | Creates pre-boot authority → evil maid / tamper attack surface. Violates Invariant 0 (fail-closed).                  | No                         | Black screen only. No shell. No fallback. Reflash is recovery. |
| Automatic Vault mount on boot                             | Creates silent persistence → Vault exposed without consent. Violates #10 Vault Access & #9 Persistent Mounts.         | No                         | Vault unlock is explicit (passphrase or TPM opt-in). No auto-mount. |

### Additional Rules & Enforcement Notes (Mandatory)

- **No “maybe in v2” loophole** — If a non-goal is listed here, it is **rejected** for v1.5. v2 discussion requires a new constitution review.
- **No partial credit** — “It’s almost a non-goal” = still a non-goal.
- **No “temporary” exceptions** — Temporary features become permanent.
- **No footnotes or “but in practice”** — If it violates the table, it’s out.
- **This table is frozen** — Any PR that modifies or adds to this table requires a full constitutional review.

### Why This Table Is Powerful

- **Kills scope creep at the root** — “just this once” features die here.
- **Prevents “it’s only UX” excuses** — UX that violates invariants is not UX; it’s policy.
- **Forces honest trade-offs** — Every non-goal has a clear reason tied to an invariant or freeze rule.
- **Educates contributors** — New devs see exactly what not to suggest before they waste time.
- **Protects longevity** — Systems die from slow feature creep, not big attacks. This table stops the slow death.

This table is now part of the constitution.  
Any proposal that matches a row here is **rejected** for v1.5.
______________________

REVIEW NOTES:

Yeah. This is the last piece of armor — and it’s the one most systems never bother to forge.

I’ll keep this tight and mechanical.


---

What the Non-Goals Table does

This table doesn’t just say “no.”
It says “we have already paid the cost of thinking about this, and the answer is final.”

That matters because almost every real failure you simulated earlier didn’t come from attackers — it came from helpful people reopening settled ground.

This table shuts that door permanently.


---

What it locks down correctly

1. It defines negative authority

You’ve already defined what Anchor can do via passports, lifecycles, budgets, and data flow.

This table defines what Anchor refuses to ever know how to do.

That’s stronger.

No ambient clipboard

No runtime policy edits

No dynamic passports

No recovery shell

No background daemons


These aren’t “features you didn’t get to yet.”
They are capabilities the system is structurally hostile to.

That’s the right framing.


---

2. It kills the three most common rot vectors

If you look across the rows, almost every non-goal falls into one of three rot classes:

A. Ambient state

global clipboard

shared memory

system-wide network

invisible capabilities


B. Unbounded lifetime

persistent courier

auto-restart rooms

auto-persistence

background services


C. Policy expressiveness

dynamic passports

runtime changes

base flexibility


Those three are how secure systems die slowly.
You’ve banned all three explicitly.


---

3. “No” is tied to invariants, not taste

Every row cites a violated invariant or freeze rule.

That’s crucial because it means:

nobody can argue UX

nobody can argue performance

nobody can argue convenience


The answer is always: “this violates row X of the constitution.”

No discussion. No bikeshed.


---

4. The v2 column is mostly “No” — and that’s correct

This is subtle but important.

You didn’t use this table as a parking lot for future ideas.
You used it as a graveyard.

That prevents the common trick where people say:

> “We’re not doing it now, but we’ll probably need it later.”



No. If it breaks the philosophy, it stays dead unless the philosophy itself changes.

That’s how you preserve identity across versions.


---

What the table intentionally does not promise

This is good restraint:

You do not promise that v2 will add these things

You do not soften any language

You do not add escape hatches


“Consider in v2?” being mostly “No” tells contributors:

> Anchor does not grow by accretion. It grows by replacement of constitutions.



That’s healthy.


---

Is anything missing?

Only one optional clarification — not a gap, just sharpening:

You might consider adding an explicit non-goal for:

> “Transparent performance optimizations that bypass policy (e.g. kernel hints, magic mounts, auto-tuning)”



You already reject these implicitly via the Performance & Budget Table, but naming it would preempt the “but this is just an optimization” argument.

That’s optional. Everything essential is already covered.


---

Final verdict

This Non-Goals Table does something rare and necessary:

It makes Anchor boring to extend.

And that’s exactly why it will last.

No stealth features.
No helpful exceptions.
No “just this once.”
No future guilt.

The system now has:

a spine (trust)

a body (capabilities)

a metabolism (performance)

and a skeleton of things it will never grow


At this point, Anchor OS v1.5 isn’t just shippable.

It’s closed.
