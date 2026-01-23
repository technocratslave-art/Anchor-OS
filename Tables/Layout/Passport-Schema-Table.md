Here is the **Passport Schema Table** — the only policy interface developers are allowed to touch in Anchor OS v1.5.

This table defines every capability a room can have.  
If a feature cannot be expressed as one or more rows in this table, **it is not a feature** — it does not ship in v1.5 (and likely never).

| Field name                  | Type       | Default     | Scope (room/session/system) | Revocation mechanism                              | Description / Enforced Invariant                                                                 | Allowed Values / Notes                                                                 |
|-----------------------------|------------|-------------|-----------------------------|---------------------------------------------------|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| `network`                   | enum       | `none`      | room                        | Close room or kill room                           | Controls network namespace policy                                                                | `none`, `wan-only`, `lan-only`, `full`                                                 |
| `gpu`                       | boolean    | `false`     | room                        | Close room or kill room                           | Whether to bind-mount NVIDIA devices (/dev/nvidia*)                                             | `true` / `false`                                                                       |
| `camera`                    | boolean    | `false`     | room                        | Close room or kill room                           | Whether to bind-mount camera devices (/dev/video*)                                              | `true` / `false`                                                                       |
| `mic`                       | boolean    | `false`     | room                        | Close room or kill room                           | Whether to bind-mount microphone devices (/dev/snd*)                                            | `true` / `false`                                                                       |
| `usb`                       | boolean    | `false`     | room                        | Close room or kill room                           | Whether to bind-mount USB devices (/dev/bus/usb*)                                               | `true` / `false`                                                                       |
| `persist`                   | array[string] | `[]`     | room                        | Close room or kill room (data survives in vault)  | List of paths inside room to bind-mount from vault subvolume (persistent)                       | e.g. `["~/.config/firefox", "~/Downloads"]`                                            |
| `shared`                    | array[object] | `[]`  | room                        | Close room or kill room                           | List of shared folders (drop-box style) with access mode                                         | `{ "name": "docs", "mode": "read-only" \| "write" }`                                   |
| `clipboard`                 | boolean    | `false`     | room                        | Close room or kill room                           | Whether to enable session clipboard (named channels)                                            | `true` / `false`                                                                       |
| `clipboard_channels`        | array[string] | `[]`  | room                        | Close room or kill room                           | Explicit list of allowed clipboard channels for this room                                       | e.g. `["work", "media"]`                                                               |
| `max_file_size_mb`          | integer    | `50`        | room                        | Close room or kill room                           | Maximum file size allowed through courier for this room                                         | Positive integer (MB)                                                                  |
| `allowed_types`             | array[string] | `[]`  | room                        | Close room or kill room                           | MIME types allowed through courier (empty = all)                                                | e.g. `["application/pdf", "image/*"]`                                                  |
| `memory_limit_gb`           | float      | `4.0`       | room                        | Close room or kill room                           | Maximum RAM the room cgroup can use                                                             | Positive float (GB)                                                                    |
| `cpu_quota_percent`         | integer    | `100`       | room                        | Close room or kill room                           | CPU quota (percentage of total host CPU)                                                        | 1–100                                                                                  |
| `pids_limit`                | integer    | `1000`      | room                        | Close room or kill room                           | Maximum number of processes/threads in the room cgroup                                          | Positive integer                                                                       |
| `auto_unlock`               | boolean    | `false`     | system (global)             | Revoke sealed secret + rotate salt                | Whether TPM auto-unlocks vault on boot (with travel mode toggle)                                | `true` / `false` (system-wide setting)                                                 |
| `travel_mode`               | boolean    | `false`     | system (global)             | Toggle + next boot forces passphrase              | Forces passphrase on next boot even if auto-unlock is enabled                                   | `true` / `false` (system-wide toggle)                                                  |

### Additional Rules & Enforcement Notes (Mandatory)

- **All fields are optional except `network` and `memory_limit_gb`** (defaults apply).
- **No field may be dynamic/scripted/conditional** — passports must be flat TOML/JSON.
- **Revocation always means:**
  - Close/kill room → all ephemeral state erased (tmpfs, clipboard, mounts, courier)
  - Auto-unlock revocation → wipe TPM secret + rotate vault key derivation salt
- **Visibility requirement:** Every non-default capability must have a visible indicator (badge, status bar icon, boot note, or explicit UI toggle).
- **Default deny:** Anything not listed in the passport is denied at spawn time.
- **No implicit allow:** Even if a field is missing, it defaults to the most restrictive value (usually `false` or `none`).

### Why This Table Prevents Creep

- Every capability is a single row → no hidden combinations
- Every row has a revocation path → no permanent traps
- Every row has a visibility requirement → no surprise features
- Flat passport constraint → no policy language explosion
- Scope is always room or system → no global ambient authority

This table is now the **only** place policy decisions are made.  
If a feature cannot be added as one or more rows here, it is **not** a v1.5 feature.
_____________________________________

Review:
This is ready to freeze. From a systems-design and governance standpoint, it’s complete, internally consistent, and—most importantly—closed under extension. There’s no place for a feature to “hide”.

Here’s a precise assessment, then a couple of micro-level checks (not changes, just confirmations you may want to codify in CI).


---

Why this schema works

1. It is the only choke point
You’ve achieved something rare: every capability funnels through one flat structure. No side channels, no “helper config”, no daemon flags. That alone kills 80% of long-term drift.

2. Flat ≠ weak here
Even though it’s flat, the schema is expressive enough because lifetime, scope, and revocation are externalized into invariants. You didn’t try to encode logic in the passport—that’s the right move.

3. Revocation semantics are uniform
This is huge. “Close/kill room” as the universal revoke for room-scoped capabilities keeps the mental model trivial and the implementation auditable.

4. Defaults are defensive, not “helpful”
Default deny + explicit allow is consistently applied. There’s no row where omission accidentally grants power.

5. System vs room scope is cleanly separated
Only two global knobs exist (auto_unlock, travel_mode), and both are explicitly system-scoped with heavyweight revocation. That keeps “global state” rare and visible.


---

Line-by-line sanity check (nothing broken)

Device access fields (gpu, camera, mic, usb)
Clean, boolean, bind-mount based. No capability creep like “read-only camera” or “temporary mic”. Good restraint.

Persistence (persist)
Correctly modeled as paths, not “enable persistence”. That prevents accidental whole-home persistence.

Shared folders
Object-based entry with name + mode is exactly the right amount of structure—no arbitrary paths, no per-file rules.

Clipboard + channels
Splitting enable (clipboard) from scope (clipboard_channels) prevents accidental global exposure. This is a subtle but important win.

Courier limits (max_file_size_mb, allowed_types)
These are security-relevant but not power-granting, which is exactly the kind of thing that belongs in the passport.

Resource limits
Explicit cgroup knobs prevent “performance surprises” and double as DoS containment.



---

Two micro-clarifications you may want to codify (optional, not rewrites)

These aren’t conceptual changes—just guardrails you might want to enforce in validation.

1. clipboard_channels requires clipboard = true
Enforce in schema validation:
If clipboard_channels is non-empty and clipboard=false → hard error.


This avoids ambiguous intent.


2. shared[].mode = write requires one-time user consent
You already state this elsewhere; just ensure it’s enforced as:

First write attempt → prompt
Prompt decision cached only for lifetime of room
Not a schema change, just implementation discipline.

---

What not to add (important)
No per-field TTLs (lifetime is structural, not declarative)
No “profiles” or presets
No inheritance or includes
No computed defaults
No “capability groups”


All of those would punch holes in the flatness guarantee.


---

Final verdict

This table, combined with:
the Invariants Table, and
the Capability Freeze Rule Table


means Anchor OS v1.5 now has:
a closed policy surface
a finite capability set
a mechanically enforceable review process
zero reliance on taste or trust


At this point, you can confidently tell dev teams:

> “If you want to add a feature, show me the row.”

If there’s no row, there’s no feature.

Locked.
