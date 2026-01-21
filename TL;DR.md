TL;DR (What problems this actually fixes)

Anchor OS exists to allow on base for Distro forks, drift, and security collapse in systems that try to be both secure and flexible.

Fork problem (primary):
Traditional “secure” systems eventually fork because people need exceptions. Once runtime mutability, plugin systems, or policy toggles exist, downstreams diverge, patches stop flowing, and security assumptions rot.

Anchor OS prevents forks by:
Making the base immutable and non-negotiable
Forcing all flexibility into rooms (not the base)
Making “just add a flag” or “just enable a feature” mechanically impossible
Turning disagreements into room-level experiments instead of base changes


If you want to experiment, you don’t fork the OS — you spawn a room.

Policy drift problem:
Most systems start locked down, then accumulate exceptions (“temporary,” “just for this app”) until the policy no longer matches reality.

Anchor OS solves this by:
Enforcing policy only at spawn time
Making policy immutable at runtime
Treating invalid or weakened policy as a fatal error, not a warning
Logging all authority crossings explicitly (courier only)


No silent degradation is possible.

Persistence problem:
In conventional systems, compromise survives reboot via filesystem writes, startup hooks, or firmware-adjacent state.

Anchor OS solves this by:
Read-only base + tmpfs rooms
Explicit, per-room persistence only
Reboot as the correct recovery action
A/B slots + Secure Boot to guarantee known-good state


If it survives reboot, it’s a bug.

Kernel 0-day realism problem:
Many systems pretend kernel exploits don’t exist or claim isolation they can’t actually provide.

Anchor OS is honest:
Kernel exploit = in-session compromise
Kernel exploit ≠ persistent compromise
Vault exposure is scoped and documented
Recovery is fast, mechanical, and boring


No security theater.

“Too secure to use” problem:
Security systems fail when they make normal workflows impossible.

Anchor OS solves this by:
Allowing rich software stacks in rooms
Letting users choose risk profiles (web vs work vs offline)
Making dangerous actions explicit instead of forbidden
Turning crashes into expected, safe outcomes


Security is visible and explainable.

What Anchor OS is not:
Not a general-purpose mutable OS
Not a sandbox bolted onto Linux
Not a microkernel research project
Not “unhackable”


What it is:
A system that stays correct over time, resists forks, survives compromise, and gives developers a place to experiment without weakening the base.

One sentence:
Anchor OS the base never changes, prevents forks and security rot by making the base immutable and pushing all flexibility into disposable rooms.


## What Is It?
An OS where the foundation is frozen (immutable) and user activity happens in disposable "rooms" that can be destroyed instantly without affecting anything else.

## Core Idea
- **Base system:** Read-only, signed, verified (never changes at runtime)
- **Rooms:** Isolated workspaces (web, work, dev) - close them, malware dies
- **Transfers:** Explicit only (no ambient clipboard/sharing)
- **Updates:** Atomic A/B (never bricks, always reversible)

## Key Properties
1. Malware cannot persist (rooms are ephemeral, base is read-only)
2. Compromised browser ≠ compromised documents (namespace isolation)
3. System never slows down (no accumulated cruft)
4. Updates are safe (watchdog auto-reverts failures)
5. Data encrypted at rest (LUKS2 + TPM)

## How It Works

```
┌─────────────────────────────────────┐
│  IMMUTABLE BASE (Kernel + Bay0)     │  ← Never changes
│  • Signed UKI                       │
│  • dm-verity verified               │
│  • No loadable modules              │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  ISOLATED ROOMS                     │
│  ┌──────┐  ┌──────┐  ┌──────┐     │
│  │ WEB  │  │ WORK │  │ DEV  │     │
│  │(temp)│  │(temp)│  │(temp)│     │  ← Disposable
│  └──────┘  └──────┘  └──────┘     │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  ENCRYPTED VAULT                    │
│  • Only persistent storage          │  ← Your data
│  • Subdivided per room              │
│  • TPM-sealed encryption            │
└─────────────────────────────────────┘
```

## User Experience

**Opening a room:**
```
Click "Web" → Browser opens (fresh)
Browse → Download file
Close → Everything gone (except file in vault)
```

**Moving files between rooms:**
```
Right-click → "Send to Work"
Prompt: "Transfer invoice.pdf to Work?"
Click "Yes" → File appears in Work
```

**What happens if browser crashes:**
```
Click "Web" again → Fresh browser
(Crash = security working, not failure)
```

## The Honest Truth

### What It Protects Against (✅)
- Persistent malware (base is read-only)
- Browser exploits (namespace isolation)
- Data leaks between contexts (explicit transfers only)
- System rot (ephemeral rooms)
- Ransomware (can't modify base)
- Failed updates (A/B rollback)

### What It Doesn't Protect Against (❌)
- Kernel 0-days *while system running* (mitigated, not prevented)
- Social engineering (user clicking "Yes")
- Physical attacks with pro tools (v0.1 basic protection only)
- Coercion (forcing passphrase disclosure)

### The Kernel 0-Day Reality
- **In-session:** Kernel exploit can compromise device while running
- **After reboot:** Exploit dead (cannot persist)
- **Mitigation:** Hardened kernel (410k LOC attack surface), fast updates (48h goal)

## Who Should Use This

**Good fit:**
- Want computer that never rots
- Browse sketchy sites safely
- Need document isolation
- Tired of malware cleanup
- Value privacy
- Use multiple Distros, better than 'Hot Swap'
- Make their own stylized Rooms (Launcher, Skins, Policies, Bricks, own voice).

**Not good fit:**
- Need cutting-edge software immediately
- Want traditional Linux flexibility
- Need broad hardware support (v0.1 is Duet 5)
- Require Windows/Mac apps

## Technical Details (Brief)

**Architecture:**
- Layer 0: UEFI Secure Boot + TPM (hardware trust)
- Layer 1: Hardened kernel + Bay0 PID 1 (~3k LOC, Rust)
- Layer 2A: Encrypted vault (LUKS2, Btrfs subvolumes)
- Layer 2B: Rooms (namespaces, tmpfs, seccomp)
- Layer 3: A/B updates (Nix, UKI, watchdog)

**Kernel hardening:**
- No loadable modules
- No BPF JIT, no io_uring, no unprivileged userns
- Lockdown mode, KASLR, W^X, stack canaries
- ~410k LOC attack surface (tracked)

**Room isolation:**
- Separate namespaces (pid, mount, net, user, ipc, uts)
- Minimal syscalls (~50 for web vs ~300 standard)
- Cgroup limits (memory, CPU, PIDs)
- No camera/mic in untrusted rooms

## Quick Start

```bash
# Build
nix build .#uki

# Sign
sbsign --key PK.key --cert PK.crt \
  --output anchor.signed.efi result/anchor.efi

# Deploy (see BUILD.md for full procedure)
```

## Performance

- Boot to usable: ~12s
- Room spawn: ~1.5s
- Room switch: ~80ms
- File transfer (10MB): ~800ms
- Update (excluding download): ~90s
- Kernel hardening overhead: <5%

## Status

- **Specification:** 100% complete
- **Documentation:** 22 documents, 100+ DON'Ts
- **Implementation:** In progress
- **ETA v0.1.0:** ~20 weeks
- **License:** TBD (GPLv3 candidate)

## The One Rule

> **"If you need runtime flexibility, you're in the wrong layer."**

Runtime flexibility = rooms (disposable)
Compile-time rigidity = base (secure)

## Comparison (Very Brief)

| System | Anchor OS | Qubes OS | NixOS | Ubuntu |
|--------|-----------|----------|-------|---------|
| Base | Immutable | Mutable | Immutable | Mutable |
| Isolation | Namespaces | VMs | None | None |
| Performance | Native | Heavy | Native | Native |
| Flexibility | Low | Medium | High | High |
| Security | High | Very High | Medium | Low |
