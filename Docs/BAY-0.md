# Bay0 — System Governor (PID 1)

Bay0 is the only mutable logic in the Anchor OS base layer.  
It is the governor — not a service manager, not a shell, not a daemon host.

Bay0 runs as PID 1 and forms the core of the Trusted Computing Base.

If Bay0 is compromised or incorrect, the entire system is considered broken.

---

## Role

Bay0 enforces invariants. It does not provide features.

Bay0:
- Boots the system
- Maintains immutability of the base layer
- Creates, isolates, and destroys rooms
- Mediates all access to persistent storage
- Mediates all inter-room data transfer
- Arms and tickles the hardware watchdog
- Halts or reboots on invariant violation

Bay0 does not:
- Render any UI
- Execute user workloads
- Provide a shell or terminal
- Manage networking
- Supervise daemons or services
- Accept runtime configuration
- Expose any mutable API beyond read-only /proc/spine

---

## Trust Model

- Bay0 is fully trusted
- All other code (kernel included) is untrusted by default
- Rooms are considered hostile
- User processes are considered hostile
- Compromise of Bay0 requires physical recovery (re-flash or rollback)

Bay0 assumes:
- The kernel may contain exploitable flaws
- Rooms will attempt privilege escalation or escape
- All inputs are potentially adversarial

---

## Process Properties

- Executes as PID 1
- Static binary strongly preferred (musl target)
- Written in Rust (memory-safe by default)
- Minimal dependency graph
- No dynamic loading, plugins, or scripting engines

If Bay0 exits or panics → kernel panic and reboot (deliberate design).

No restart mechanism exists above Bay0.

---

## Filesystem Mounts (Bay0 Namespace)

Bay0 manages the Spine namespace:

- Mounts Spine SquashFS → `/` (ro,loop)
- Mounts devtmpfs → `/dev`
- Mounts procfs → `/proc`
- Mounts sysfs → `/sys`
- Mounts tmpfs → `/run`
- Bind-mounts encrypted vault → `/persist` (nodev,nosuid,noexec)

Bay0 is the only process permitted to call mount().

Rooms:
- Cannot invoke mount()
- Cannot observe parent mounts
- Cannot enumerate or traverse `/persist` root
- Cannot execute binaries from persistent storage

---

## /proc/spine (Read-Only Control Plane)

Bay0 exposes a minimal, read-only interface at `/proc/spine/`:

Example files:
- `/proc/spine/version`
- `/proc/spine/uptime`
- `/proc/spine/rooms` (list of active rooms)
- `/proc/spine/watchdog`
- `/proc/spine/health`

Properties:
- Strictly read-only
- No writes allowed from any process
- No ioctls or control operations
- No sensitive data exposed

---

## Room Lifecycle

### Spawn Sequence (Strict Order)

1. Clone new namespaces: mount, pid, net, ipc, uts, user
2. Create cgroup slice with hard limits (memory.max, cpu.max, pids.max)
3. Mount room `.sqsh` image read-only in private mount namespace
4. pivot_root into room filesystem
5. Bind only explicitly allowed `/persist` subpaths (nodev,nosuid,noexec; no vault root access)
6. Apply hardening:
   - Strict seccomp filter
   - Resource limits (rlimits)
   - no_new_privs
   - Empty capability bounding set
7. execve() room init binary

Any failure aborts spawn. No partial or retry logic.

---

### Runtime Isolation Guarantees

Rooms cannot:
- Observe or signal other rooms
- Signal Bay0
- Mount filesystems
- Load kernel modules
- Modify embedded policies
- Access devices unless explicitly granted by passport

All isolation enforced by kernel primitives:
- Namespaces
- Cgroups
- Seccomp
- Capabilities

Bay0 never trusts room behavior.

---

### Exit & Cleanup

On room termination (normal exit, crash, or kill):
- Cgroup terminates entire process tree
- All processes reaped
- Mount namespace destroyed
- Room image unmounted
- Ephemeral state discarded

No persistent residue. No recovery needed.

---

## Courier (Mediated Data Transfer)

Bay0 exclusively handles inter-room data movement.

Transfer flow:
1. Source room requests transfer (via AF_UNIX socket to Bay0)
2. Bay0 authenticates peer via SO_PEERCRED
3. Source room frozen
4. Payload validated (size cap, type, policy lattice check)
5. User confirmation required for uphill trust moves (Bay0 framebuffer prompt)
6. Spawn one-shot courier child process
7. Stream data via pre-opened pipes (bounded size & time)
8. Courier exits immediately after transfer
9. Log transfer (source, dest, size, hash, decision)
10. Unfreeze source room

Courier constraints:
- Empty namespaces (no fs, no net)
- Strict seccomp (read/write/close/exit only)
- Max lifetime 5 seconds
- Never reused or persisted

No shared clipboard, shared memory, or ambient channels allowed.

---

## Policy (Passport) Handling

Policies are:
- Parsed during Nix build
- Embedded in UKI/initrd
- Immutable at runtime

Bay0:
- Loads policies once at startup
- Refuses boot on parse or schema failure
- Refuses room spawn on policy violation

Rooms:
- Cannot read or query policies
- Only discover limits by attempting forbidden actions

No runtime policy changes or external fetches.

---

## Watchdog & Fail-Closed Behavior

Bay0:
- Arms hardware watchdog at startup
- Tickle periodically
- Proves liveness

If Bay0 stops tickling (hang, deadlock, crash):
- Hardware watchdog triggers full system reset
- Boot returns to known-good state (A/B slot rollback if configured)

This is intentional fail-closed design.

---

## Logging & Audit

Bay0 logs:
- Room spawn / exit
- Courier transfers
- Policy violations
- Seccomp denials
- Watchdog events
- Fatal errors

Destinations:
- `/run/log` (volatile tmpfs)
- Optional encrypted append-only log in `/persist/system/audit`

Bay0 never logs:
- User data payloads
- Vault contents
- Secrets

---

## Hard Security Constraints

Bay0 must never:
- Exceed 3000 LOC (excluding tests)
- Use network sockets beyond AF_UNIX local
- Render graphics or handle input devices
- Load plugins or dynamic code
- Execute from `/persist`
- Accept runtime flags or config beyond cmdline
- Silence panics
- Continue after invariant violation

Any violation triggers immediate halt or reboot.

---

## Dependency Rules

- Minimal audited dependencies
- Avoid unsafe Rust code
- No network-related crates
- No heavy filesystem abstractions

Enforcement:
- Max 10 direct dependencies
- New dependency requires security review
- `unsafe` blocks require justification
- CI fails on unauthorized changes

---

## Failure Semantics

| Failure Type                 | Response                        |
|------------------------------|---------------------------------|
| Panic                        | Kernel panic & reboot           |
| Policy parse/schema error    | Refuse to boot                  |
| Watchdog timeout             | Hardware reset                  |
| Mount or spawn failure       | Abort room, log, continue       |
| Security invariant violation | Immediate reboot                |
| Courier bounds violation     | Kill courier, abort transfer, log; repeated → reboot |

No graceful degradation. No partial operation.

---

## Testing Requirements

- Unit tests: policy parsing, seccomp filters, cgroup setup, courier bounds
- Integration tests: QEMU boot, room isolation, vault scoping, watchdog reset
- Negative tests: mount/ptrace/capability escapes, invalid policies, resource exhaustion, adversarial /proc/spine inputs

Tests that violate invariants are considered passing.

---

## Mental Model

Bay0 is:
- The bus driver
- The jailer
- The fuse

It is not a platform, not a framework, not an extensible service layer.

Flexibility, features, and convenience belong in rooms — never in Bay0.

---

## Non-Negotiables

- Bay0 is PID 1
- Bay0 is small and boring
- Bay0 never recovers from failure
- Bay0 never negotiates invariants
- Bay0 never grows features

The base layer does not change.  
Bay0 exists to enforce that single truth.
