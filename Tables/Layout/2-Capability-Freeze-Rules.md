**Capability Freeze Rules Table** — the mechanical gate for Anchor OS v1.5.

Every proposed feature, relaxation, or PR **must** pass **all five** requirements.  
If **any** cell is **No**, the proposal is **rejected** for v1.5 (no exceptions, no partial credit, no “defer”, no “mostly fits”).  
This replaces taste, debate, and social pressure with five binary checks.

| Requirement (The Gate)                          | Definition / What It Means                                                                 | Yes / No | If “No” → Rejected (no exceptions in v1.5) |
|-------------------------------------------------|--------------------------------------------------------------------------------------------|----------|---------------------------------------------|
| 1. Explicit Capability Handle                   | Feature must have a named, scoped, revocable handle (e.g. “session-clipboard”, “shared-drop-box”) — not ambient or global | Yes      | No named handle = ambient authority = rejected |
| 2. Visible On/Off State                         | Feature must have clear, always-visible state (indicator, badge, boot note, status bar entry) — user must never wonder if it’s on | Yes      | Invisible state = surprise = rejected |
| 3. Revoke Cheaper than Enable                   | Revocation (clear, disable, toggle off) must be cheaper/faster than enabling (one-click, one-toggle, no re-auth) | Yes      | Hard-to-revoke = trap = rejected |
| 4. Bounded Lifetime                             | Feature must have automatic end condition (session end, timeout, room close, reboot) — no unbounded or “forever” lifetimes | Yes      | Unbounded = infrastructure = rejected |
| 5. Fits Flat Passport Declaration               | Feature must be expressible in flat passport TOML (no conditionals, no scripting, no dynamic rules) — simple key-value only | Yes      | Expressive policy = rot = rejected |

### How to Use This Gate (Operational Instructions)

1. Any proposal (new feature, relaxation, UX change) must be written as:
   - Name/handle
   - Passport field (if any)
   - Lifetime / scope
   - Enable/revoke flow
   - Visibility mechanism

2. Apply the table row-by-row.

3. If **all five** are **Yes** → Allowed in v1.5 (and must be implemented exactly as described).  
   If **any** is **No** → Rejected for v1.5 (move to v2 discussion or discard).

### Examples (real proposals, real outcomes)

| Proposal / Feature                              | Handle | Visible State | Revoke Cheaper | Bounded Lifetime | Flat Passport | Overall | Reason (if No) |
|-------------------------------------------------|--------|---------------|----------------|------------------|---------------|---------|----------------|
| Session Clipboard (named channels)              | Yes    | Yes           | Yes            | Yes              | Yes           | **Yes** | — |
| Shared Folder (drop box, read-only default)     | Yes    | Yes           | Yes            | Yes              | Yes           | **Yes** | — |
| Per-Room Persistent Mounts (e.g. ~/.config)     | Yes    | Yes           | Yes            | Yes              | Yes           | **Yes** | — |
| Boot Auto-Unlock (TPM-sealed passphrase)        | Yes    | Yes           | Yes            | Yes              | Yes           | **Yes** | — |
| Ambient Clipboard (global, no indicator)        | No     | No            | No             | No               | No            | **No**  | Violates all 5 — ambient authority |
| Runtime Policy Change in Running Room           | No     | No            | No             | No               | No            | **No**  | Violates all 5 — escalation vector |
| Global Shared Memory / IPC                      | No     | No            | No             | No               | No            | **No**  | Violates all 5 — covert channel |
| Arbitrary Mount Points for Shared Folders       | No     | No            | No             | No               | No            | **No**  | Violates #1, #5 — path confusion |
| Auto-Persistence of All Room State              | No     | No            | No             | No               | No            | **No**  | Violates #3, #4 — silent persistence |
| Dynamic / Scripted Passports (conditionals)     | No     | No            | No             | No               | No            | **No**  | Violates #5 — policy rot |
| Persistent Courier (reusable connection)        | No     | No            | No             | No               | No            | **No**  | Violates #4 — persistent channel |
| Base Runtime Flexibility (live policy edit)     | No     | No            | No             | No               | No            | **No**  | Violates #1, #5 — drift risk |

### Why This Table Prevents v2 Creep

- **Binary checks** — no “partially fits” wiggle room  
- **No social override** — taste, popularity, “it’s just UX” don’t matter  
- **No “defer” loophole** — rejected means rejected for v1.5  
- **All requirements orthogonal** — satisfying 4/5 still fails  
- **Every “Yes” is enforceable** — passport field, indicator, revocation flow, lifetime bound

This table is the gate.  
It replaces endless design discussions with five yes/no questions.  
It keeps Anchor OS from slowly becoming “just another OS.”
