**Capability → Indicator Mapping Table** — a strict requirement for Anchor OS v1.5.

**Rule:** Every capability must have a visible, always-present indicator.  
No exceptions.  
No hidden toggles.  
No “advanced users can disable indicators.”  
If a capability lacks a row here with a defined indicator, **it is invalid** and **does not ship** in v1.5.

| Capability                          | Indicator Shown                                      | Where Visible                              | Auto-Clear / Revocation Condition                          | Notes / Enforcement Rationale |
|-------------------------------------|------------------------------------------------------|--------------------------------------------|------------------------------------------------------------|-------------------------------|
| Session Clipboard (named channels)  | Small badge/icon: “Clipboard active (from Web)”      | Room chrome / status bar (top-right)       | Room close, session timeout (5 min), explicit “Clear now”  | Visible per-channel source; prevents surprise paste |
| Shared Folder (drop box)            | Badge/icon: “Shared folder active (Docs)”            | Room chrome / status bar                   | Room close, explicit unmount (one-click)                   | Shows current shared folders; read-only default |
| Per-Room Persistent Mounts          | Badge/icon: “Persistent storage active”              | Room chrome / status bar                   | Room close, explicit unmount (one-click)                   | Shows which paths are persistent; prevents confusion |
| Boot Auto-Unlock (TPM-sealed)       | Boot splash text: “Auto-unlocked (TPM)” + “Travel mode OFF” | Boot splash / lock screen                  | Revoke sealed secret (one-click) + next boot forces passphrase | Must be visible at boot; travel mode toggle shown |
| Ambient Clipboard (session-only)    | Prominent indicator: “Ambient clipboard ON (session)” + “Clear now” button | Room chrome / status bar (red/orange)      | Room close, session timeout (5 min), explicit “Clear now”  | High-visibility because ambient is riskier |
| GPU Access                          | Badge/icon: “GPU enabled”                            | Room chrome / status bar                   | Room close or kill room                                    | Shows GPU is active (heat/battery impact) |
| Camera Access                       | Badge/icon: “Camera active” (red dot style)          | Room chrome / status bar (red)             | Room close or kill room                                    | Privacy-critical; red dot standard |
| Mic Access                          | Badge/icon: “Mic active” (red dot style)             | Room chrome / status bar (red)             | Room close or kill room                                    | Privacy-critical; red dot standard |
| USB Access                          | Badge/icon: “USB devices active”                     | Room chrome / status bar                   | Room close or kill room                                    | Shows external device risk |
| Network Access (any mode)           | Badge/icon: “Network active (WAN-only)”              | Room chrome / status bar                   | Room close or kill room                                    | Shows connectivity level; WAN-only is most common |

### Additional Rules (Mandatory)

- **Indicators must be persistent** while capability is active — no auto-fade, no “advanced hide” option.
- **Revocation must be one-click or one-toggle** from the indicator itself (e.g. click badge → “Revoke / Clear / Disable”).
- **Color coding** for risk: red for camera/mic, orange for clipboard/shared, green/blue for normal (GPU, network).
- **Boot splash must show** any system-wide capabilities (e.g. auto-unlock, travel mode).
- **No capability without an indicator** — if it can’t be shown, it can’t exist.

### Why This Table Prevents Invisible State

- Every capability has a **visual anchor** — users see exactly what’s active.
- Every revocation is **visible and cheap** — users can undo without digging.
- The system is **self-documenting** — no need to read docs to know what’s on.
- No “stealth mode” loophole — invisible power = ambient authority = forbidden.

This table is now part of the constitution.  
Any capability without a row here is **invalid by definition**.
__________________________________

Review Notes:

This is correct, complete, and enforceable. You’ve closed one of the last classic escape hatches: invisible power. At this point the system can’t surprise the user even if someone tries to be clever.

Here’s a tight assessment and the only optional refinements worth considering.

Why this table works:

Indicators are treated as security primitives, not UX polish. That’s the key win. You’ve elevated visibility to the same level as seccomp or mount flags.

Revocation is colocated with visibility. This kills the “I know it’s on, but where do I turn it off?” failure mode that destroys trust.

Risk-weighted signaling is explicit. Red/orange/green isn’t cosmetic; it encodes threat level at a glance and aligns with existing user instincts (camera/mic red dots).

System-wide state is surfaced at boot, not buried in settings. That’s rare and absolutely correct for something like auto-unlock.


Row-by-row sanity check:

Session clipboard + ambient clipboard separation is excellent. You’re acknowledging different risk classes and demanding different visual weight.

Shared folders and persistent mounts are clearly distinct, which avoids the common “why is this file still here?” confusion.

GPU/network indicators pull double duty: security and performance transparency. That’s smart.

Camera/mic indicators match established OS norms, which reduces cognitive load and avoids relearning.


The “Additional Rules” section is doing real constitutional work:

“No auto-fade” and “no advanced hide” explicitly block the usual UX-driven erosion that happens later.

“Indicator must exist or feature doesn’t exist” is the cleanest possible rule. No exception process means no slow rot.


Two optional micro-clarifications (not required, just belt-and-suspenders):

1. Indicator presence is enforced at capability enable time
You might add one line to enforcement rationale:



“Capability enable fails hard if indicator cannot be rendered.”


This prevents headless or misconfigured UIs from accidentally creating invisible state.

2. Multiple indicators must stack, not collapse
If several capabilities are active, indicators should accumulate rather than merge into a single “room has privileges” icon.



This preserves granularity and avoids ambiguity.

Again: optional. The intent is already clear.

What you should not add:

No severity scores

No “expert mode”

No indicator suppression

No contextual hiding (“only show when active use detected”)


All of those reintroduce surprise.

Final verdict:

With this table in place, Anchor OS now guarantees something most systems only promise:

> If something has power, it is visible.
If it is visible, it is revocable.
If it is revocable, it is bounded.



That’s the end of invisible state.

This table is constitution-grade.
Lock it.

