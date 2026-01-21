TL;DR

Anchor OS is an immutable, signed base system that runs all user activity inside isolated, disposable “rooms.”

The base never changes at runtime. If something breaks, crashes, or is compromised, you reboot or reset the room. Nothing persists unless explicitly allowed.

Each room is isolated with namespaces, cgroups, and seccomp. Rooms cannot talk to each other directly. All cross-room data movement goes through a one-shot courier that validates, logs, and then disappears. No clipboard, no drag-and-drop, no ambient IPC.

The kernel is heavily hardened and treated honestly: a kernel 0-day can compromise the system in-session, but it cannot persist. Rebooting kills the exploit. The base remains intact, updates are atomic, and a known-good state is always recoverable via A/B slots and Secure Boot.

Persistent data lives in an encrypted vault split into per-room subvolumes. Rooms only see the one subvolume they are allowed to mount. Sensitive data can be kept in “cold storage” that is never auto-mounted and only accessed in short-lived, network-none rooms.

Security is enforced mechanically:

No kernel modules

Minimal kernel attack surface

Explicit syscall allowlists

CI-enforced LOC and config limits

Fast kernel updates (48h goal)

Reboot as the incident response


What Anchor OS guarantees:

No persistence after compromise

No cross-room contamination

No base modification

Clean rollback after failure


What it does not promise:

Kernel exploits never happen

In-session data is always safe if the kernel is compromised


Core rule:
If you need runtime flexibility, you’re in the wrong layer.

Anchor OS trades convenience for containment, recovery, and long-term correctness.
