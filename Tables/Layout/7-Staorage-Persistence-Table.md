Here is the **Storage & Persistence Table** — the enforcement layer that guarantees Anchor OS remains "forgetful by default" in v1.5.

This table defines every storage type that can hold data, its lifetime, scope, default state, and wipe conditions.  
**Rule:** If a storage type is not in this table, **it does not exist**.  
No unlisted persistence. No silent state. No exceptions.

| Storage Type                  | Lifetime                          | Scope (room / system) | Default State       | Wipe Conditions                                                                 | Extra Info / Enforcement Notes |
|-------------------------------|-----------------------------------|-----------------------|---------------------|----------------------------------------------------------------------------------|--------------------------------|
| tmpfs overlay (room runtime)  | Room lifetime                     | room                  | Always ephemeral    | Room close, kill room, reboot                                                    | All runtime files, caches, processes — erased from RAM on death. No disk footprint. |
| Session clipboard (named channels) | Session lifetime                  | room                  | Off                 | Room close, session timeout (5 min), explicit "Clear now"                        | Named channels prevent global ambient leak. Visible indicator required. |
| Shared Drop Box (read-only)   | Room lifetime                     | room                  | Off                 | Room close, explicit unmount                                                     | Read-only default. Append-only write from source room. Fixed mount points. |
| Shared Drop Box (write-enabled) | Room lifetime                     | room                  | Off                 | Room close, explicit unmount                                                     | Write requires one-time consent + passport flag. No overwrite allowed. |
| Per-Room Persistent Mount (vault subvolume) | Until revoked or deleted          | room                  | Off                 | Explicit unmount, revoke consent, room delete, vault wipe                        | Bind-mount from /vault/sub/<room-id>/. One-time prompt + visible indicator. Scoped to room ID. |
| Vault root (/vault)           | Until explicit wipe or passphrase change | system                | Sealed              | Explicit wipe, passphrase change, TPM seal revocation                            | LUKS2 + TPM-sealed. Never mounted globally into rooms. Only subvolumes exposed. |
| Host export (/etc/anchor/host) | Boot lifetime                     | system → room         | Read-only           | Reboot (unmounted)                                                               | Metadata only (e.g. driver version.json). No secrets. No authority. RO bind-mount. |
| Courier dropzone (/vault/drop) | Until explicit promote or delete  | room → system         | Ephemeral (until promote) | Room close, explicit delete, promote to Vault                                    | Write-only from rooms. Courier promotes to models/ subvolume only. Audit logged. |
| Model weights (Vault models/) | Until explicit delete or wipe     | system                | Persistent          | Explicit delete, vault wipe, passphrase change                                   | Read-only bind-mount into rooms. Never writable from rooms. Courier promote only. |

### Additional Rules & Enforcement Notes (Mandatory)

- **No unlisted storage** — Any storage not in this table is forbidden. No hidden tmpfs, no /tmp hacks, no shared /dev/shm.
- **Default is ephemeral / off** — Persistence and sharing are opt-in only. No silent save.
- **Wipe is deterministic** — Every storage type has a hard end condition. No "maybe later" or "user forgets."
- **Revocation is cheap** — One-click unmount, revoke consent, or kill room clears almost everything. Vault wipe is the nuclear option.
- **Visibility is mandatory** — Persistent storage and shared folders require visible indicators (status bar badge, boot note, etc.).
- **No cross-scope leak** — Room-scoped storage cannot be seen by other rooms or the host unless explicitly couriered.

### Why This Table Enforces "Forgetful by Default"

- **Everything is ephemeral unless explicitly persistent** — tmpfs is the default for all runtime.
- **Persistence is always scoped** — room ID in subvolume path prevents cross-room access.
- **No silent accumulation** — No caches, no logs, no history unless user says yes.
- **Wipe is automatic and complete** — Room close = RAM wipe. Reboot = full ephemeral reset.
- **Audit trail is bay0-only** — No room can see or modify logs.

This table is now part of the constitution.  
Any storage not listed here **does not exist** in v1.5.
_____________________________
REVIEW NOTES:

This table lands exactly where it needs to. It’s consistent with the prior four tables, closes a real class of failures, and—most importantly—it does not smuggle in new power under the guise of “storage details.”

I’ll do this in three passes:

1. what the table says (and enforces),


2. what it implicitly forbids (which is just as important),


3. what it doesn’t say and whether anything should be added.



I’ll be blunt where needed.


---

1. What the Storage & Persistence Table says

A. Storage is a closed set, not a filesystem accident

The strongest property here is the rule:

> If a storage type is not in this table, it does not exist.



That turns storage from “whatever Linux gives you” into constitutional scope. This aligns perfectly with the Data Flow Table and the Lifecycle Table: storage is not infrastructure, it’s a capability.

This immediately kills:

hidden caches

implicit logs

surprise persistence via libraries

“but /tmp is always there” thinking


That’s a major win.


---

B. Ephemerality is the default, mechanically enforced

The table is ruthless about defaults:

tmpfs overlay → always ephemeral

clipboard → off

shared folders → off

persistence → off

vault → sealed


There is no storage that quietly exists.

This directly enforces:

Invariant #5 (Room Persistence)

Freeze Rule #4 (Bounded Lifetime)

Lifecycle Table (no zombies)


And it does so without relying on user discipline or documentation. The system forgets by construction.


---

C. Every storage type has a deterministic death

This is where the table shines.

Every row answers: how does this die?

Room close → RAM wiped

Session timeout → clipboard wiped

Explicit unmount → shared folders gone

Revoke consent → persistence gone

Reboot → host exports gone

Passphrase change → vault unreadable


There is no “garbage collection later,” no “user forgot,” no “we’ll clean it up eventually.”

This is exactly what most secure systems get wrong. You didn’t.


---

D. Vault is not a filesystem, it’s a resource

Two very important things are explicit:

1. /vault is never mounted wholesale


2. Only subvolumes are exposed, and only by bind-mount



That keeps the Vault from becoming:

a shared namespace

a de facto home directory

a sideways IPC channel


The split between:

models/ (RO, system-scoped)

sub/<room-id>/ (scoped persistence)

dropzone/ (write-only staging)


is exactly the right tri-partition. It matches the War Room fixes and closes the Lazy Courier class permanently.


---

E. Visibility and revocation are first-class

You explicitly tie storage to:

visible indicators

one-click revoke paths


That matters because storage is the hardest thing to mentally model for users. By forcing indicators, you prevent the classic “why is this still here?” moment that kills trust.

This is consistent with:

Indicator Mapping Table

Freeze Rule #2 and #3



---

2. What the table forbids implicitly (this is crucial)

This table quietly but decisively bans a lot of dangerous “normal Linux” behavior.

A. No shared /dev/shm, no anonymous hugepages

Because it’s not listed, it doesn’t exist.

That closes:

shared memory IPC

side-channel persistence

accidental cross-room leaks


Given your GPU + hugepage discussion earlier, this is important: hugepages must be room-scoped tmpfs or not exist. This table enforces that.


---

B. No log persistence in rooms

There is no row for:

application logs

shell history

crash dumps


Which means:

logs belong to bay0 only

rooms leave no forensic residue by default


That’s a conscious design choice, and it matches “forgetful by default.” Anyone who wants persistent logs must opt into persistence explicitly—and visibly.


---

C. No “performance cache” loopholes

You explicitly block:

RAM caches pretending to be persistence

write-back caches that dodge indicators

“it’s just a cache, not storage” nonsense


This closes one of the nastiest real-world footguns (your intern optimization attack from earlier).


---

D. No cross-room visibility via storage

Because:

subvolumes are room-ID–scoped

dropzones are write-only from rooms

shared folders are explicit and fixed


There is no way to “discover” another room’s data by browsing storage. Discovery itself would be an ambient channel; you’ve eliminated it.


---

3. What the table does not say (and whether it should)

This is the important part. There are only two areas worth even discussing, and neither is a flaw.

1. GPU memory residency (intentional omission, good)

You do not list GPU VRAM as a storage type.

That’s correct.

Why:

VRAM is volatile

lifetime is tied to process / device reset

persistence would be a hardware bug, not policy


By not listing it, you implicitly say:

> “If it survives a room death, that’s a bug.”



That’s the right stance. Do not add VRAM to this table.


---

2. Swap / NVMe-backed paging (borderline, but okay as-is)

Given your earlier discussion of pinned NVMe swap for huge models, someone might ask:

> “Where does swap live in this table?”



Your current answer is implicit:

swap is an implementation detail of the room runtime

lifetime = room lifetime

wipe condition = room close / reboot


That is acceptable as long as:

swap files are created inside the room tmpfs namespace or a room-scoped scratch area

they are never reused across rooms


If you want to be extra explicit (optional), you could add:

> “Room-local swap backing files are treated as tmpfs overlay data and wiped on room death.”



But you don’t strictly need a new row. The current table already forbids persistent swap by omission.


---

Final verdict

This Storage & Persistence Table:

is consistent with all prior tables

closes real attack classes discovered in the War Room

does not introduce new authority

does not leak convenience back in

is mechanically enforceable


Most importantly, it preserves the core psychological promise of Anchor OS:

> Nothing stays unless you asked for it, and you can always see what stayed.



At this point, the constitution is complete:

Capabilities are gated

Lifecycles are bounded

Data flows are enumerated

Storage is finite and forgetful


Anything else added after this is either:

v2 research

or a violation you can point to by row number and say “no.”
