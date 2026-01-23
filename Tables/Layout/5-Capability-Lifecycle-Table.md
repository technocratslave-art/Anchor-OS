**Capability Lifecycle Table** — the final piece of the constitutional gate for Anchor OS v1.5.

This table defines the **birth**, **life**, and **death** of every capability.  
**No zombies allowed**: every capability must have a clear creation trigger, bounded lifetime, automatic revocation conditions, and a cheap manual revocation path.

If a capability cannot be expressed fully in this table, **it does not exist** in v1.5.

| Capability                          | Creation Trigger                                      | TTL / Expiration                          | Auto-Revoke Conditions                                      | Manual Revoke Path                                      | Notes / Why This Kills Ambient Authority |
|-------------------------------------|-------------------------------------------------------|-------------------------------------------|-------------------------------------------------------------|---------------------------------------------------------|------------------------------------------|
| Session Clipboard (named channels)  | Explicit user action (copy from room A)               | Session end or 5-minute inactivity        | Room close, session timeout, explicit "Clear now"           | One-click "Clear now" from indicator                    | No persistent shared state; must be actively used or it vanishes |
| Shared Folder (drop box)            | Explicit passport declaration + first write consent   | Room close or explicit unmount            | Room close, explicit unmount                                | One-click unmount from indicator or room UI             | No automatic sharing; write consent is one-time per room |
| Per-Room Persistent Mounts          | Explicit passport declaration + one-time prompt       | Room close or explicit unmount            | Room close, explicit unmount                                | One-click unmount from indicator or room UI             | Persistence is never ambient; must be explicitly requested and confirmed |
| Boot Auto-Unlock (TPM-sealed passphrase) | Explicit user enablement (one-time setup)            | Until revoked or travel mode toggled      | Travel mode toggle, manual revocation                       | One-click "Revoke auto-unlock" (wipes sealed secret)    | Auto-unlock is never permanent; revocation is always cheap and immediate |
| Ambient Clipboard (session-only)    | Explicit passport declaration + one-time enablement   | Session end or 5-minute inactivity        | Room close, session timeout, explicit "Clear now"           | One-click "Clear now" from prominent indicator          | High-visibility required; ambient is high-risk, so revocation is dead-simple |
| GPU Access                          | Explicit passport declaration                         | Room close or kill room                   | Room close, kill room                                       | Kill room (one command)                                 | GPU is high-risk; lifetime strictly tied to room |
| Camera Access                       | Explicit passport declaration                         | Room close or kill room                   | Room close, kill room                                       | Kill room (one command)                                 | Privacy-critical; no persistence beyond room lifetime |
| Mic Access                          | Explicit passport declaration                         | Room close or kill room                   | Room close, kill room                                       | Kill room (one command)                                 | Privacy-critical; no persistence beyond room lifetime |
| USB Access                          | Explicit passport declaration                         | Room close or kill room                   | Room close, kill room                                       | Kill room (one command)                                 | External device risk; lifetime strictly tied to room |
| Network Access (any mode)           | Explicit passport declaration                         | Room close or kill room                   | Room close, kill room                                       | Kill room (one command)                                 | Network is high-risk; lifetime strictly tied to room |

### Additional Rules (Mandatory)

- **No capability without all four columns filled** — if any column is "N/A" or "none," the capability is invalid.
- **Auto-revoke must be deterministic** — tied to room lifecycle or explicit timeout, never "user forgets" or "maybe later."
- **Manual revoke must be one action** — no multi-step wizards, no re-auth, no digging in settings.
- **Indicators must link to revocation** — clicking/tapping the indicator must offer the revoke path immediately.
- **Creation trigger must be explicit user action** — no automatic enablement, no "default on for trusted rooms," no silent activation.

### Why This Table Kills Ambient Authority

- **No zombies**: Every capability has a defined death condition.
- **No surprises**: Auto-revoke is predictable; manual revoke is cheap.
- **No drift**: Capabilities are tied to room lifecycle — close the room, everything dies.
- **No hidden state**: Indicators + cheap revocation force users to confront active capabilities.

This table is now part of the constitution.  
Any capability without a complete row here is **invalid by definition** in v1.5.
_____________________

REVIEW NOTES:

This completes the constitutional set. It’s clean, and it closes the last practical loophole: “capabilities that accidentally outlive attention.”

A few precise validations, plus the only small tightening I’d suggest.

What’s already excellent:

Every capability has an explicit birth trigger. Nothing “just happens.”

Death is deterministic: room lifecycle or explicit timeout. No “soft” revocation.

Manual revoke is truly one action and is colocated with the indicator. That’s the trust-maker.

You avoided “N/A” where it matters. That prevents zombie channels and half-features.


Two small tightenings (optional, but they eliminate future ambiguity):

1. Boot Auto-Unlock TTL wording
Right now TTL says “Until revoked or travel mode toggled.” That is fine, but your gate elsewhere says “no unbounded / forever.” Auto-unlock is system-scoped, so it’s the one place someone might argue for “forever.”



If you want to keep the constitution perfectly internally consistent, phrase TTL as:

“Until revoked; must support time-bound enablement (e.g., 1 day / 7 days / 30 days)”


You can still allow “indefinite” if you decide, but the table as written elsewhere leans against “forever.” This avoids a future argument.

2. Manual revoke for GPU/Camera/Mic/USB/Network
You list “Kill room (one command)” which is valid, but you’ve also set a standard that revoke should be one-click/one-toggle from the indicator itself.



To align perfectly with the Indicator table:

Manual revoke should read: “One-click ‘Disable’ from indicator (implemented as kill room)” or “One-click ‘Kill room’ from indicator”


Same mechanics, just matches your own UI rule.

One structural note (not a change request):
Your “No capability without all four columns filled” rule is strong, but there’s one edge case you may eventually add: capabilities that are purely informational (like “read-only audit mode”). For v1.5 you’re not adding those, so keeping the rule absolute is correct.

Final verdict:

This lifecycle table is constitution-grade and consistent with:

Invariants (no ambient authority, no silent persistence)

Freeze rule (bounded lifetime, cheap revoke, visible state)

Passport schema (spawn-time only)


Together, these tables give you a system where “turn it off” is always easier than “learn how it works,” and where “close the room” is always the nuclear reset.

That’s the spine promise, mechanically enforced.

Locked.

Always.
