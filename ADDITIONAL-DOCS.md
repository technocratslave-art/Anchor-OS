# Anchor OS - Additional Documentation

## Table of Contents

- [For Users](#for-users)
- [For Developers](#for-developers)
- [For Security Researchers](#for-security-researchers)
- [Architecture Deep Dive](#architecture-deep-dive)
- [Comparison to Other Systems](#comparison-to-other-systems)
- [Real-World Usage Scenarios](#real-world-usage-scenarios)
- [Troubleshooting](#troubleshooting)
- [Roadmap](#roadmap)


## For Users

### What is Anchor OS?

Anchor OS is an operating system designed around one core idea: **the parts that matter never change, and the parts that might break are disposable.**

Think of it like a house with rooms:
- The foundation (walls, roof, plumbing) never changes
- Each room is separate and can be cleaned instantly
- If something breaks in one room, other rooms are fine
- You can't accidentally damage the foundation while using a room

### Who Should Use Anchor OS?

**Good fit:**
- You want a computer that never slows down or gets "gunked up"
- You browse sketchy websites but want to stay safe
- You work with sensitive documents and need real isolation
- You're tired of malware persistence and cleaning infections
- You value privacy and want minimal data leakage
- You want security without becoming a security expert

**Not a good fit (yet):**
- You need cutting-edge software immediately (we audit before adding)
- You need to run Windows/macOS-only software
- You need kernel-level access for development (use dev room, but it's isolated)
- You want traditional desktop flexibility (this is intentionally rigid)

### Day-to-Day Usage

**Opening a room:**
```
1. Click "Web" icon on dock
2. Browser opens (fresh, clean)
3. Browse normally
4. Close when done
→ Next time you open: completely clean slate
```

**Moving files between rooms:**
```
1. Download file in Web room
2. Right-click → "Send to Work"
3. Confirm prompt: "Send invoice.pdf to Work?"
4. Click "Yes"
→ File appears in Work room's inbox
```

**What happens if Web room crashes?**
```
Nothing bad!
- Click Web icon again
- Fresh browser opens
- Old crash is gone
- No cleanup needed
```

This is **security working correctly**, not a failure.

### Understanding the Security Model

**What you're protected from:**
- ✅ Malware cannot survive reboot
- ✅ Compromised browser cannot access your documents
- ✅ Bad website cannot steal files from other rooms
- ✅ Ransomware cannot encrypt your vault (base is read-only)
- ✅ Failed updates cannot brick your device
- ✅ Physical theft leaves your data encrypted and unreadable

**What you're NOT protected from:**
- ❌ Social engineering (you clicking "Yes" on bad prompts)
- ❌ Coercion (someone forcing you to unlock)
- ❌ Sophisticated kernel exploits (while system is running)
- ❌ Physical access with professional tools (evil maid with hardware)

**The honest boundary:** If someone exploits the kernel while your vault is open, they might see the files you currently have open. But they cannot persist the exploit, cannot modify the system, and rebooting kills the infection completely.

### Common User Questions

**"Why did my room crash?"**

Usually because it tried to do something the security system blocked. This is **protection**, not failure. Just reopen the room.

**"Why can't I copy-paste between rooms?"**

v0.1 doesn't have clipboard because clipboard is an "invisible pipe" that malware can use. You must explicitly send files via "Send to..." (which shows a confirmation).

v0.2 will add a better clipboard that still requires confirmation.

**"Why is my web browser older than on my friend's Ubuntu?"**

We audit new browser updates for new kernel syscalls before allowing them. This takes 2-4 weeks. You're safer, but sometimes behind.

**"Can I install random software?"**

In Dev room: yes, full freedom.
In Web/Work rooms: no, these are curated.

This trade-off keeps those rooms secure.

**"What if I forget my vault passphrase?"**

Your data is unrecoverable. This is intentional - it means thieves also cannot recover it. **Write down your passphrase and store it safely.**



## For Developers

### Development Philosophy

Anchor OS is **not** a traditional Linux distribution. It's a security substrate with a different mental model.

**Traditional Linux:**
```
System = mutable base + runtime configuration + services
Security = permissions + policies + updates
Flexibility = edit config files, install packages
```

**Anchor OS:**
```
System = immutable base + disposable contexts
Security = physical isolation + cryptographic verification
Flexibility = rebuild base, spawn new contexts
```

**Key mindset shift:** You don't "configure" Anchor OS at runtime. You **define** it at build time, then deploy immutable snapshots.

### Developer Workflow

**Typical day:**

```bash
# Morning: Pull latest
git pull origin main

# Make changes to bay0
cd bay0/src
vim room.rs
cargo test

# Check LOC limit
tokei bay0/src -t=Rust -o json | jq '.Rust.code'
# Must be < 3000

# Build new UKI locally
nix build .#uki-debug  # Debug build for testing

# Test in QEMU
./scripts/test-uki.sh result/anchor.efi

# If good, submit PR
git add bay0/
git commit -m "Improve room spawn error handling"
git push origin feature/better-errors

# CI runs:
# - LOC check
# - All tests
# - Integration tests
# - Security checks

# After review + approval: merged
# Nightly build creates production UKI
```

**You never SSH into Anchor OS to "fix" it.** You fix the source code, rebuild, redeploy.

### Adding a New Feature

**Example: Add new syscall to web room**

```bash
# 1. Justify it
vim docs/SYSCALL-JUSTIFICATIONS.md
```

```markdown
## memfd_create

**Date:** 2026-01-25
**Reason:** Firefox 128 requires it
**Risk:** Medium (kernel memory subsystem)
**Alternatives:** None (required by Firefox)
**Decision:** Approved after review
```

```bash
# 2. Update policy
vim policies/rooms/web.toml
# Add "memfd_create" to syscalls.allow

# 3. Update attack surface budget
vim docs/ATTACK-SURFACE-BUDGET.md
# Document LOC impact

# 4. Test
nix build .#uki
./tests/syscall-test.sh web memfd_create

# 5. Submit with review checklist
# Requires security team + kernel expert approval
```

**Timeline:** 2-3 days for approval + testing + merge

### Working with Bay0

**Bay0 is special.** It's PID 1. If it crashes, kernel panics. Rules:

```rust
// ✅ GOOD: Explicit error handling
pub fn spawn_room(policy: &RoomPolicy) -> Result<RoomHandle> {
    let pid = create_namespace(policy)?;
    apply_cgroups(pid, policy)?;
    Ok(RoomHandle { pid })
}

// ❌ BAD: Unwrap in bay0 (will panic)
pub fn spawn_room(policy: &RoomPolicy) -> RoomHandle {
    let pid = create_namespace(policy).unwrap();  // NEVER DO THIS
    apply_cgroups(pid, policy).unwrap();
    RoomHandle { pid }
}

// ✅ GOOD: Panic only for truly unrecoverable errors
pub fn main() {
    mount_essentials().expect("Cannot mount /proc - unrecoverable");
    // This is acceptable: if /proc fails, system is unusable anyway
    
    loop {
        match handle_event() {
            Ok(_) => continue,
            Err(e) => {
                log::error!("Non-fatal error: {}", e);
                continue;  // Keep running
            }
        }
    }
}
```

**LOC budget is sacred:**

```bash
# Before every commit
tokei bay0/src -t=Rust -o json | jq '.Rust.code'

# If > 3000:
# - Refactor to remove code
# - Extract to separate component
# - Simplify logic
# - Remove feature

# NEVER:
# - Ask to raise limit
# - Disable check temporarily
# - "It's just 50 lines"
```

**Why 3000 LOC?**

- Can be fully read in one sitting (~8 hours)
- Can be audited by external reviewer in one week
- Small enough to hold mental model
- Large enough for actual functionality

If you need more, you're building the wrong thing in bay0.

### Testing Your Changes

**Unit tests:**
```bash
cd bay0
cargo test
```

**Integration tests:**
```bash
# Boot test
./tests/qemu-boot-test.sh result/anchor.efi

# Room isolation test
./tests/room-isolation-test.sh

# Courier test
./tests/courier-test.sh
```

**Fuzzing (before release):**
```bash
# 72-hour minimum
./tests/fuzz-syscalls.sh
```

**Manual testing:**
```bash
# Flash to test device
./scripts/deploy-to-slot.sh anchor-test.signed.efi test-device

# Test:
# - Boot successfully
# - Spawn all rooms
# - Transfer files between rooms
# - Update to new slot
# - Rollback on failure
```

### Common Mistakes (and How to Avoid)

**Mistake 1: Adding runtime flexibility**
```rust
// ❌ BAD
if config.allow_unsafe_mode {
    disable_seccomp();
}

// ✅ GOOD
// Build two separate UKIs: production (secure) and debug (flexible)
// Never mix them
```

**Mistake 2: Storing secrets in bay0**
```rust
// ❌ BAD
static MASTER_KEY: &[u8] = b"secret123";

// ✅ GOOD
// Keys live in TPM or vault, never in code
let key = tpm_unseal()?;
```

**Mistake 3: Creating ambient channels**
```rust
// ❌ BAD
// Shared memory between rooms
let shm = SharedMemory::new("/tmp/shared");
room_a.map(shm);
room_b.map(shm);

// ✅ GOOD
// Explicit courier transfer
courier::transfer(room_a, room_b, file_fd)?;
```

**Mistake 4: Ignoring failure modes**
```rust
// ❌ BAD
room.spawn().ok();  // Silent failure

// ✅ GOOD
match room.spawn() {
    Ok(handle) => log::info!("Room spawned: {}", handle.pid),
    Err(e) => {
        log::error!("Room spawn failed: {}", e);
        // Decide: retry? abort? fallback?
    }
}
```

### Performance Optimization (with Constraints)

**Rule:** Never optimize at the cost of security.

```rust
// ❌ BAD: Skip validation for speed
pub fn transfer(src: RawFd, dst: RawFd) -> Result<()> {
    // Fast but unsafe
    copy_unchecked(src, dst)?;
    Ok(())
}

// ✅ GOOD: Validate, then optimize within constraints
pub fn transfer(src: RawFd, dst: RawFd, max_size: u64) -> Result<()> {
    validate_magic_bytes(src)?;  // Security first
    validate_size(src, max_size)?;
    
    // THEN optimize the copy
    let mut buf = [0u8; 65536];  // Larger buffer = faster
    loop {
        let n = read(src, &mut buf)?;
        if n == 0 { break; }
        write_all(dst, &buf[..n])?;
    }
    Ok(())
}
```

**Profile before optimizing:**

```bash
# Profile bay0
cargo flamegraph --bin bay0

# Identify hotspots
# Optimize only proven bottlenecks
# Verify security preserved
```


## For Security Researchers

### Threat Model

**Assumptions (what we trust):**
- Hardware vendor (CPU, TPM are not backdoored)
- UEFI firmware (basic functionality, Secure Boot enforcement)
- Rust compiler (not malicious)
- User (not actively malicious, but may make mistakes)

**Out of scope (v0.1):**
- Nation-state attackers with physical access + professional tools
- Supply chain attacks on hardware/firmware
- Side-channel attacks (timing, spectre, etc.)
- Social engineering / coercion

**In scope (what we defend against):**
- Opportunistic malware
- Targeted malware (medium sophistication)
- Browser exploits (including 0-days)
- Kernel exploits (partial mitigation)
- Physical theft without professional tools
- Evil maid (basic level)

### Attack Surface Analysis

**Total trusted computing base:**
- Kernel: ~410k LOC (hardened, no modules)
- Bay0: ~3k LOC (Rust, minimal)
- Compositor: ~30k LOC (isolated persona)
- Total: ~443k LOC

**High-risk components:**
- Kernel networking stack (~80k LOC)
- Kernel filesystems (~190k LOC)
- Browser (in room, ~5M LOC but isolated)

**Attack vectors (prioritized by feasibility):**

| Vector | Feasibility | Impact | Mitigation |
|--------|-------------|--------|------------|
| Browser exploit | High | Medium | Namespace isolation, ephemeral |
| Kernel 0-day | Medium | High | Attack surface reduction, fast updates |
| Physical theft | Medium | Low | LUKS2 + TPM sealing |
| Social engineering | High | Medium | Explicit prompts, user education |
| Supply chain | Low | High | Signature verification, reproducible builds |
| Evil maid (advanced) | Low | High | Not mitigated in v0.1 (v0.2: Boot Guard) |

### Known Vulnerabilities (v0.1)

**By design (honest boundaries):**

1. **Kernel 0-day in-session exposure**
   - If kernel exploited while vault mounted → data in that subvolume exposed
   - Mitigation: Reboot kills exploit, fast updates (48h goal)
   - Cannot eliminate: kernel is shared substrate

2. **GPU side-channel timing**
   - Shared GPU may leak timing information between rooms
   - Mitigation: None in v0.1 (low priority)
   - v0.2: Explore GPU isolation if hardware supports

3. **Physical ESP reflash**
   - ESP partition not measured by TPM
   - Attacker with tools + physical access can reflash
   - Mitigation: Physical security
   - v0.2: Boot Guard + tamper seals

4. **Coercion**
   - User can be forced to unlock vault, approve transfers
   - Mitigation: Emergency wipe, user education
   - Cannot solve with software

**Bugs (will fix):**
- TBD (report to security@anchor-os.example)

### Exploit Scenarios (Red Team)

**Scenario 1: Browser RCE → Room Compromise**

```
1. User visits malicious site in Web room
2. Site exploits browser vulnerability
3. Attacker gains code execution in browser process
4. Attacker attempts privilege escalation
   → Blocked by seccomp (limited syscalls)
5. Attacker attempts to read vault
   → Blocked by namespace (vault not mounted or not visible)
6. Attacker attempts to access other rooms
   → Blocked by PID namespace (cannot see other rooms)
7. Attacker attempts to persist
   → Room is tmpfs, reboot/close kills all state
8. Attacker attempts to exfiltrate downloads
   → Limited by max_file_size, logged
   
OUTCOME: Contained to web room session
RECOVERY: Close room (or reboot)
```

**Scenario 2: Kernel RCE → Device Compromise (In-Session)**

```
1. Attacker exploits kernel vulnerability via web room
2. Attacker gains kernel code execution
3. Attacker can now:
   ✓ Read mounted vault subvolumes
   ✓ Access all running rooms
   ✓ Exfiltrate data over network
   ✗ Modify base system (dm-verity prevents)
   ✗ Persist exploit (base is read-only)
   ✗ Survive reboot (tmpfs cleared)
4. User reboots (either manual or automatic)
5. Exploit killed, system clean

OUTCOME: In-session compromise, no persistence
RECOVERY: Reboot + credential rotation
```

**Scenario 3: Physical Theft → Data Extraction**

```
1. Device stolen while powered off
2. Attacker attempts to boot
   → UEFI requires custom keys (not present)
3. Attacker attempts to read vault
   → LUKS2 encrypted, no passphrase
4. Attacker attempts TPM unsealing
   → TPM requires PCR match (system not booted with correct UKI)
5. Attacker attempts to extract TPM key
   → Would require chip-level tools (v0.1 doesn't prevent)
6. Attacker attempts to brute-force passphrase
   → Argon2id + strong passphrase = infeasible

OUTCOME: Data unreadable (unless advanced tools used)
RECOVERY: Remote wipe if device phones home, or data lost
```

### Fuzzing Targets

**Priority 1 (high risk):**
- Syscalls allowed in web room
- Courier transfer validation
- Policy parser (TOML → seccomp)

**Priority 2 (medium risk):**
- Bay0 umbilical socket handler
- Vault mount logic
- Namespace creation

**Priority 3 (low risk):**
- Update verification
- Watchdog tickle

**Recommended tools:**
- syzkaller (kernel syscalls)
- AFL++ (user-space parsers)
- cargo-fuzz (Rust components)

### Responsible Disclosure

If you find a vulnerability:

1. **Do NOT open public issue**
2. Email: security@anchor-os.example (PGP encouraged)
3. Include:
   - Vulnerability description
   - Reproduction steps
   - Impact assessment
   - Suggested fix (optional)
4. Expected response: 48 hours
5. Coordinated disclosure: 90 days
6. Credit: Hall of Fame (with permission)

**Bug bounty:** Not available for v0.1 (considering for v0.2)


## Architecture Deep Dive

### Why Four Layers?

**Layer 0 (Hardware Trust):** Without this, attacker can modify system before OS loads. TPM + Secure Boot create root of trust.

**Layer 1 (Bay0):** Single control plane prevents confused deputy attacks. If multiple components could spawn rooms, they could disagree about policy.

**Layer 2A (Vault):** Persistent data needs its own layer with different threat model than ephemeral rooms.

**Layer 2B (Rooms):** Execution contexts need isolation from each other and from base.

**Layer 3 (Lifecycle):** Updates need to be atomic and reversible, orthogonal to runtime operation.

Each layer has **exactly one responsibility** and **clear guarantees**.

### Why No Kernel Modules?

**Security:** 90% of kernel vulnerabilities historically involved modules. Static kernel = huge attack surface reduction.

**Determinism:** Module load order can affect behavior. Static kernel = reproducible.

**Verification:** Can hash entire kernel. With modules, state changes at runtime.

**Trade-off:** Cannot add drivers without rebuilding kernel. Acceptable for targeted hardware (Duet 5).

### Why Rust for Bay0?

**Memory safety:** PID 1 crash = kernel panic. Memory safety critical.

**No GC:** Garbage collection pauses unacceptable for PID 1.

**Zero-cost abstractions:** Performance competitive with C.

**Modern tooling:** cargo, clippy, rust-analyzer make development faster.

**Trade-off:** Larger binary than C (acceptable), smaller ecosystem (manageable).

### Why Nix?

**Reproducibility:** Same input → same output (bit-for-bit).

**Atomic rollbacks:** Never overwrite, always create new version.

**Cross-compilation:** Build ARM on x86 trivially.

**Supply chain:** Lock files pin exact versions.

**Trade-off:** Learning curve steep (acceptable), not universally adopted (manageable).

### Design Decisions (Rationale)

**Why tmpfs for rooms?**
- Ephemeral by default prevents persistence
- Fast (RAM-backed)
- Automatically cleared on close
- Alternative (overlay on disk) allows persistence

**Why SquashFS for base images?**
- Read-only by design
- Good compression
- Fast random access
- Mature, stable

**Why courier process instead of shared memory?**
- Shared memory = ambient channel
- Process = clear lifetime
- Process = auditable
- Process = killable if hangs

**Why explicit prompts instead of automatic policy?**
- User must understand data flow
- No hidden transfers
- Simple mental model
- Prevents confused deputy

**Why no clipboard?**
- Clipboard = ambient IPC
- Any room can read without asking
- v0.2: Visual escrow (explicit, one-shot)


## Comparison to Other Systems

### vs. Qubes OS

**Similarities:**
- Both use isolation (rooms / qubes)
- Both have explicit inter-VM communication
- Both target security-conscious users

**Differences:**

| Feature | Anchor OS | Qubes OS |
|---------|-----------|----------|
| Base | Immutable (read-only) | Mutable |
| Rooms/VMs | Lightweight (namespaces) | Heavy (full VMs) |
| Performance | Native | Virtualization overhead |
| Persistence | Explicit (vault) | Per-VM persistence |
| Updates | Atomic A/B | Template-based |
| Complexity | Simpler (1 kernel) | Complex (Xen + multiple kernels) |
| Hardware | Specific (Duet 5 initially) | Broad |

**When to choose Qubes:** Need full VM isolation, run multiple OSes, have powerful hardware

**When to choose Anchor:** Want simplicity, performance, immutability, targeted hardware

### vs. NixOS

**Similarities:**
- Both use Nix for declarative config
- Both are reproducible
- Both can roll back

**Differences:**

| Feature | Anchor OS | NixOS |
|---------|-----------|-------|
| Mutability | Immutable base | Mutable |
| Isolation | Rooms (namespaces) | None (single OS) |
| Updates | A/B atomic | Generation-based |
| Modules | No kernel modules | Loadable modules |
| Security focus | Core design goal | Configuration option |
| Flexibility | Low (intentional) | High |

**When to choose NixOS:** Need traditional Linux flexibility, want declarative config, less security-focused

**When to choose Anchor:** Want immutability + isolation, willing to trade flexibility for security

### vs. Fedora Silverblue / openSUSE MicroOS

**Similarities:**
- Immutable base
- Atomic updates
- Container-based applications

**Differences:**

| Feature | Anchor OS | Silverblue/MicroOS |
|---------|-----------|-------------------|
| Isolation | Kernel-level (namespaces) | Container (podman/toolbox) |
| Kernel | Hardened, no modules | Standard |
| Updates | A/B | OSTree |
| Communication | Courier (explicit) | Shared filesystem |
| Vault | Encrypted, subdivided | User home dir |

**When to choose Silverblue/MicroOS:** Want immutable Fedora/openSUSE with familiar apps

**When to choose Anchor:** Want deeper isolation, hardened kernel, explicit communication

### vs. ChromeOS

**Similarities:**
- Immutable base
- Verified boot
- Automatic updates
- Simple UX

**Differences:**

| Feature | Anchor OS | ChromeOS |
|---------|-----------|----------|
| Ownership | User owns keys | Google owns keys |
| Privacy | No telemetry | Google telemetry |
| Flexibility | Dev room available | Developer mode |
| Apps | Native Linux | Web apps + Android |
| Server | User-controlled | Google services |

**When to choose ChromeOS:** Want simplicity, Google integration, broad hardware support

**When to choose Anchor:** Want user ownership, privacy, no cloud dependency

### vs. Traditional Linux (Ubuntu/Debian/Arch)

**Why NOT traditional Linux:**
- Mutable base (rot accumulation)
- Persistent malware possible
- Complex attack surface
- Update failures can brick
- No isolation between apps
- Ambient clipboard/IPC

**Why traditional Linux might be better:**
- Mature ecosystem
- Broad hardware support
- Extensive documentation
- Large community
- More software available

**Anchor OS is NOT a replacement for traditional Linux.** It's a different tool for different use cases.


## Real-World Usage Scenarios

### Scenario 1: Journalist

**Profile:**
- Handles sensitive sources
- Browses potentially hostile websites
- Needs strong isolation
- Cannot afford data leaks

**Anchor OS setup:**
```
Rooms:
- Web (untrusted): Research, general browsing
- Work-offline: Sensitive documents, no network
- Vault-access: Ultra-sensitive sources (cold storage)

Workflow:
1. Research in Web room
2. Transfer vetted articles to Work-offline
3. Write article offline (no network = no exfiltration)
4. Access sensitive sources only when needed (vault-access)
5. Transfer final article to Web for upload

Security benefit:
- Malware in Web cannot access sensitive documents
- Work offline = no remote kernel exploits
- Cold storage minimizes exposure window
```

### Scenario 2: Developer

**Profile:**
- Writes code daily
- Needs stable system
- Experiments frequently
- Doesn't want to brick system

**Anchor OS setup:**
```
Rooms:
- Web: Stack Overflow, documentation
- Dev: Code, compilers, experiments
- Work: Final projects, deployment scripts

Workflow:
1. Research solutions in Web
2. Experiment in Dev (break things, reset quickly)
3. Move working code to Work for deployment
4. Deploy from Work (stable, tested)

Security benefit:
- Experiments cannot break base system
- Dev room is sink (toolchain contamination contained)
- Work room is trusted output only
```

### Scenario 3: Small Business Owner

**Profile:**
- Handles customer data (PII)
- Not technical
- Needs compliance (GDPR, etc.)
- Simple workflow

**Anchor OS setup:**
```
Rooms:
- Web: Email, banking, general use
- Work: Customer database, invoices
- Compliance report: Run quarterly

Workflow:
1. Daily business in Web (email, etc.)
2. Customer data only in Work
3. Generate compliance reports quarterly
4. Demonstrate isolation to auditor

Security benefit:
- Customer data isolated from web browsing
- Malware from phishing cannot access customer DB
- Compliance easier to demonstrate (architectural)
```

### Scenario 4: Security Researcher

**Profile:**
- Analyzes malware
- Visits hostile sites
- Needs safe sandbox
- Quick reset essential

**Anchor OS setup:**
```
Rooms:
- Web-disposable: One-time hostile site visits
- Analysis: Malware analysis tools
- Write-up: Final reports
- Vault-access: Archived samples (cold storage)

Workflow:
1. Visit hostile site in Web-disposable
2. Capture sample, close Web (malware dies)
3. Analyze in Analysis room
4. Write report in Write-up room
5. Archive sample to cold storage

Security benefit:
- Malware cannot persist (ephemeral)
- Analysis room isolated from write-up
- Can safely trigger malware (contained)
```


## Troubleshooting

### System Won't Boot

**Symptom:** Black screen, no bay0 message

**Diagnosis:**
1. Check Secure Boot keys enrolled correctly
2. Verify UKI signature: `sbverify --cert PK.crt anchor.efi`
3. Boot with `loglevel=7` to see kernel messages
4. Check UEFI boot order

**Solution:**
```bash
# Re-enroll keys in UEFI
# Or boot from recovery media:
dd if=anchor-recovery.efi of=/dev/mmcblk0p1
reboot
```

### Vault Won't Unlock

**Symptom:** "Failed to unlock vault" message

**Diagnosis:**
1. Passphrase incorrect (most common)
2. PCR mismatch (kernel changed)
3. TPM malfunction

**Solution:**
```bash
# Check PCR values
tpm2_pcrread sha256:0,2,4,7

# Compare to sealed values
tpm2_policypcr -l sha256:0,2,4,7

# If mismatch: kernel was changed
# Need to re-seal vault to new kernel
```

**Recovery:**
```bash
# Boot with recovery UKI (same kernel as when sealed)
# Or use backup passphrase (if set during setup)
```

### Room Won't Spawn

**Symptom:** "Failed to spawn room" error

**Diagnosis:**
```bash
# Check bay0 logs
journalctl -u bay0 | tail -50

# Common causes:
# - Invalid policy (TOML syntax)
# - Missing base image
# - Namespace creation failed (kernel config)
# - Cgroup v2 not mounted
```

**Solution:**
```bash
# Validate policy
toml-lint policies/rooms/web.toml

# Check base image exists
ls -lh /usr/lib/anchor/images/web.sqsh

# Verify kernel config
zcat /proc/config.gz | grep CONFIG_NAMESPACES
# Should show =y
```

### Courier Transfer Fails

**Symptom:** "Transfer failed" notification

**Diagnosis:**
```bash
# Check audit log
grep "courier" /vault/system/audit.log | tail -20

# Common causes:
# - Validation failed (wrong file type)
# - Policy denied (destination not allowed)
# - Timeout (file too large)
# - Size limit (exceeds max_file_size)
```

**Solution:**
- Check file type matches policy allowed_types
- Verify destination in allowed_destinations list
- Check file size against max_file_size limit
- For large files: increase timeout or use external media

### System Performance Degraded

**Symptom:** Slow room operations, lag

**Diagnosis:**
```bash
# Check memory pressure
free -h

# Check if room hitting limits
cat /sys/fs/cgroup/room-web/memory.current
cat /sys/fs/cgroup/room-web/memory.max

# Check storage
df -h
```

**Solution:**
```bash
# Close unused rooms
# Clear old vault snapshots
btrfs subvolume delete /vault/.snapshots/old-*

# Increase room memory limit (rebuild UKI)
vim policies/rooms/web.toml
# memory_limit = "6G"  # was 4G
```

### Update Failed

**Symptom:** System reverted to old version after update

**Diagnosis:**
```bash
# Check watchdog log
journalctl | grep watchdog

# Likely cause: new UKI failed health check
# Watchdog auto-reverted to old slot
```

**Solution:**
```bash
# Check build logs for new UKI
# Test new UKI in QEMU before deploying
./tests/qemu-boot-test.sh new-anchor.efi

# If test passes but deployment fails:
# Check hardware-specific issue (drivers, etc.)
```


## Roadmap

### v0.1.0 (Current) - Foundation

**Status:** Specification complete, implementation in progress

**Features:**
- ✅ Immutable base with dm-verity
- ✅ Bay0 PID 1 governor
- ✅ Room isolation (namespaces)
- ✅ Courier model (explicit transfers)
- ✅ A/B atomic updates
- ✅ Hardened kernel (no modules, minimal attack surface)
- ✅ Web, Work, Dev rooms
- ✅ TPM + LUKS2 vault

**Limitations (v0.1):**
- No clipboard (explicit transfers only)
- Limited hardware support (Duet 5 initially)
- Basic evil maid protection (UEFI Secure Boot only)
- No GPU isolation (shared GPU)

**ETA:** ~20 weeks from start of implementation


### v0.2.0 - Enhanced Security

**Focus:** Address v0.1 limitations, improve physical security

**Planned features:**
- Visual escrow clipboard (explicit, one-shot, user-approved)
- Boot Guard integration (ESP tampering detection)
- Tamper-evident seals (hardware)
- Improved evil maid detection
- GPU isolation (if hardware supports)
- Plausible deniability (hidden

vault)
- Additional hardware support (2-3 more ARM devices)
- Performance optimizations

**New capabilities:**
- Clipboard with visual confirmation (Mirror + Handle + Courier model)
- Physical tamper detection (logs + user notification)
- Hidden vault option (deniable encryption)
- Per-room GPU contexts (if supported by hardware)

**Security improvements:**
- Boot Guard prevents ESP modification
- Tamper seals detect physical opening
- Hidden vault provides plausible deniability under coercion
- GPU isolation prevents timing side-channels

**ETA:** 6-9 months after v0.1.0 release


### v0.3.0 - Research & Expansion

**Focus:** Explore advanced security, broader hardware

**Research topics:**
- Microkernel evaluation (seL4, others)
- Formal verification of bay0
- Confidential computing (AMD SEV, Intel TDX)
- Hardware security modules (beyond TPM)
- Network anonymity integration (Tor)

**Potential features:**
- Microkernel option (reduced TCB)
- Formally verified core components
- TEE support for sensitive operations
- HSM integration for enterprise
- Tor/VPN integration at base layer
- x86_64 support (laptops, desktops)

**Expansion:**
- Server/headless mode (no GUI)
- IoT/embedded variant (minimal)
- Enterprise management tools
- Multi-device sync (encrypted)

**ETA:** 12-18 months after v0.2.0


### Beyond v0.3.0 - Long-term Vision

**Potential directions:**

**1. Hardware partnerships**
- Custom hardware with Anchor OS pre-installed
- Hardware-level tamper detection
- Custom secure elements
- Open hardware designs

**2. Ecosystem development**
- Third-party room templates
- App store (curated, signed)
- Room sharing marketplace
- Developer tools

**3. Enterprise features**
- Centralized management
- Policy enforcement across fleet
- Compliance reporting
- Integration with existing tools

**4. Research OS**
- Formal verification of entire stack
- Novel isolation techniques
- Post-quantum cryptography
- Zero-knowledge proofs for privacy

**Timeline:** 2+ years, depends on adoption


## Community

### Getting Involved

**For users:**
- Join Matrix: #anchor-os:matrix.org
- Subscribe: announce@anchor-os.example
- Report bugs: https://github.com/anchor-os/anchor-os/issues
- Documentation: https://docs.anchor-os.example

**For developers:**
- Read: [docs/DEVELOPER-GUIDE.md](docs/DEVELOPER-GUIDE.md)
- Contributing: [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md)
- Code review: Participate in PRs
- Testing: Run on hardware, report issues

**For security researchers:**
- Responsible disclosure: security@anchor-os.example
- Vulnerability reports welcome
- Credit in Hall of Fame
- Future bug bounty program

**For sponsors:**
- Hardware donations (test devices)
- Development funding
- Security audits
- Documentation improvements

### Project Governance

**Current status:** Early stage, small core team

**Decision making:**
- Architecture decisions: Core team consensus
- Security decisions: Security team veto power
- Feature additions: Proposal → review → implementation
- Breaking changes: Require extensive discussion + user notice

**Core principles (non-negotiable):**
1. Honesty about limitations
2. User ownership (keys, data)
3. No telemetry without opt-in
4. Open source (code, docs, designs)
5. Security over convenience
6. Simplicity over features

**As project grows:**
- Formal governance structure (TBD)
- Foundation or similar entity (considered)
- Community voting on non-security features
- Transparent decision logs


## Philosophy & Principles

### Design Philosophy

**Core belief:** Most OS security problems stem from **ambient privilege** and **mutable state**.

**Ambient privilege:** Processes can do things without explicit permission
- Example: Any process can read clipboard
- Solution: No clipboard, or explicit escrow

**Mutable state:** System changes over time, accumulates cruft
- Example: Installed packages, config files, cached data
- Solution: Immutable base, ephemeral contexts

**The Anchor OS answer:**
1. **Immutable base:** Foundation never changes at runtime
2. **Explicit permissions:** Every boundary crossing requires explicit action
3. **Ephemeral contexts:** Throw away the room, not the data
4. **Mechanical enforcement:** Not policy, but physics

### Guiding Principles

**1. Honesty over marketing**
- Document limitations clearly
- No "unhackable" claims
- Explain trade-offs
- Share failure modes

**2. Simplicity over features**
- Fewer components = fewer bugs
- Clear mental model
- Easy to audit
- Resist feature creep

**3. Mechanical over policy**
- Enforce with code, not rules
- Hard boundaries, not soft
- Cannot be misconfigured
- No "user error" security

**4. User ownership**
- User owns keys, not vendor
- No forced updates
- No telemetry without opt-in
- Open source everything

**5. Fail secure, not convenient**
- When in doubt, deny
- Explicit > implicit
- Safe > fast
- Boring > clever

### Why These Design Choices?

**Why immutable base?**

Traditional mutable OS:
```
Day 1:   Clean system
Day 30:  +100 packages, +500 config files
Day 90:  +2000 cached files, unknown state
Day 365: Slow, fragile, full of cruft
Solution: "Reinstall" (lose everything)
```

Anchor OS:
```
Day 1:   Clean system
Day 365: Identical to Day 1 (base unchanged)
Cruft:   Lives in tmpfs, dies on close
Solution: Close room (2 seconds)
```

**Why explicit transfers?**

Traditional clipboard:
```
1. Copy in malicious browser
2. Malware reads clipboard (silent)
3. Malware knows: passwords, private keys, sensitive data
4. No user awareness
```

Anchor OS courier:
```
1. Select "Send to Work"
2. Prompt: "Transfer invoice.pdf to Work?"
3. User sees: filename, source, destination
4. User decides: Yes/No
5. Logged: source, dest, hash, timestamp
```

User always knows when data crosses boundaries.

**Why namespaces over VMs?**

VMs (Qubes OS approach):
```
Pros: Strong isolation, multiple kernels
Cons: Heavy (RAM, CPU), slow, complex
```

Namespaces (Anchor OS approach):
```
Pros: Lightweight, fast, simple
Cons: Shared kernel (but hardened)
Trade-off: Acceptable for most threats
```

If you need VM-level isolation, use Qubes. If you want performance + simplicity, use Anchor.

**Why no kernel modules?**

Modules:
```
Pros: Flexibility (add drivers without reboot)
Cons: 90% of kernel CVEs, runtime attack surface
```

Static kernel:
```
Pros: Massive attack surface reduction, predictable
Cons: Must rebuild to add driver
Trade-off: Worth it for security, acceptable for targeted hardware
```

### What Anchor OS Is NOT

**Not a desktop replacement** (yet)
- Lacks broad hardware support
- Missing many apps
- Different workflow

**Not for everyone**
- Requires learning new model
- Less flexible than traditional Linux
- Some software won't work

**Not a silver bullet**
- Cannot stop social engineering
- Cannot prevent coercion
- Cannot stop determined attacker with physical access

**Not trying to be everything**
- Focused on security + simplicity
- Specific use cases
- Opinionated design

### The Honest Assessment

**What Anchor OS does well:**
- ✅ Prevents persistent malware (ephemeral rooms)
- ✅ Contains browser exploits (namespace isolation)
- ✅ Never accumulates cruft (immutable base)
- ✅ Updates safely (A/B atomic)
- ✅ Protects offline data (encryption + TPM)
- ✅ Provides clear security model (explicit boundaries)

**What Anchor OS doesn't do well (v0.1):**
- ❌ Support broad hardware (targeted initially)
- ❌ Prevent kernel 0-days (can only mitigate)
- ❌ Protect against physical attacks (basic only)
- ❌ Support legacy software (curated apps only)
- ❌ Provide instant software updates (audit delay)

**Is this the right OS for you?**

Ask yourself:
- Do I value security over convenience?
- Am I willing to learn a new model?
- Is my threat model opportunistic malware + basic physical security?
- Can I accept some software lag (2-4 weeks for updates)?
- Do I have or can I get supported hardware?

If **yes to most:** Anchor OS might be a good fit.

If **no to many:** Traditional Linux or Qubes OS might be better.


## Frequently Asked Questions (Extended)

### General Questions

**Q: Is Anchor OS free?**

A: Yes, open source (license TBD, likely GPLv3 or similar). Free to use, modify, distribute. No paid features, no telemetry, no ads.

**Q: Who develops Anchor OS?**

A: Currently a small team. Goal: community-driven project. Seeking contributors, sponsors, and users.

**Q: Is this production-ready?**

A: v0.1 will be beta quality (use at own risk). v0.2 targets production use. Enterprise support TBD.

**Q: Can I trust this?**

A: Source code is open. Builds are reproducible (Nix). You can audit everything. We encourage independent security audits. We're honest about limitations.

**Q: What about telemetry?**

A: None by default. If added in future, opt-in only, with clear disclosure of what's collected.

**Q: Can I contribute?**

A: Yes! See [docs/DEVELOPER-GUIDE.md](docs/DEVELOPER-GUIDE.md) and [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md)

### Technical Questions

**Q: Why AArch64 first, not x86_64?**

A: 
1. Simpler platform (fewer legacy components)
2. Good power efficiency (mobile/embedded)
3. Strong hardware options (modern ARM SoCs)
4. x86_64 support planned for v0.3

**Q: Can I run Docker/Podman inside rooms?**

A: Dev room: Yes (full root, nested namespaces)
Web/Work rooms: No (too permissive)

**Q: What about Wayland/X11?**

A: Wayland only (simpler, more secure). No X11 support planned.

**Q: Can I use proprietary NVIDIA drivers?**

A: v0.1: No (requires loadable modules)
v0.2: Possibly (if we can bake into UKI)
Alternative: Use Intel/AMD integrated graphics

**Q: Does this work on Raspberry Pi?**

A: Not currently (different hardware). Possible future target if there's demand.

**Q: Can I dual-boot with another OS?**

A: Technically yes, but not recommended. Secure Boot keys + TPM sealing make it complex. Dedicated hardware recommended.

**Q: What filesystems are supported?**

A: Base: squashfs (read-only)
Vault: btrfs (primary), ext4 (possible)
Not supported: NTFS, FAT32 (for vault)

**Q: Can I encrypt swap?**

A: No swap partition (security risk). Use cgroup memory limits instead.

**Q: What about hibernation?**

A: Not supported (security risk: RAM contents written to disk). Use suspend-to-RAM only.

### Security Questions

**Q: How do I know the UKI isn't backdoored?**

A: 
1. Build from source (Nix provides reproducible builds)
2. Compare hash with community builds
3. Audit source code (open source)
4. Independent security audits (planned)

**Q: What if my TPM is compromised?**

A: TPM compromise requires physical access + professional tools. If concerned:
1. Use strong vault passphrase (TPM + passphrase = 2-factor)
2. Physical security (tamper seals in v0.2)
3. Consider external HSM (v0.3)

**Q: Can the NSA/[agency] break this?**

A: Honest answer: Probably, if targeted.
- They likely have kernel 0-days we don't know about
- They may have TPM/UEFI vulnerabilities
- They can coerce passphrase disclosure
- They have professional evil maid tools

Anchor OS defends against opportunistic attackers, not nation-states (yet).

**Q: What about Spectre/Meltdown?**

A: Kernel includes mitigations (KPTI, retpoline, etc.). Not immune, but mitigated. Room isolation helps contain exploitation.

**Q: Can I use this for cryptocurrency wallet?**

A: Vault-access room (network-none) is suitable. But:
- Consider hardware wallet (better)
- We don't claim "unhackable"
- Use at your own risk

**Q: Is this FIPS-140 certified?**

A: No (certification costs $$$$). Crypto primitives are from Linux kernel (widely trusted). Enterprise version may pursue certification.

**Q: What about GDPR/HIPAA/SOC2 compliance?**

A: Architecture helps (isolation, logging, encryption), but compliance requires more than just OS. Consult with compliance expert.

### Usage Questions

**Q: Can I install Steam/games?**

A: Dev room: Yes (but performance may vary)
Dedicated gaming room: Possible custom room policy
Not recommended: Games in Web/Work rooms (too permissive)

**Q: What about Windows apps (via Wine)?**

A: Technically possible in Dev room. Not guaranteed to work. VMs might be better for Windows apps.

**Q: Can I use this as my daily driver?**

A: Depends on your workflow:
- Web browsing: Yes (optimized for this)
- Office work: Yes (LibreOffice, etc.)
- Software development: Yes (Dev room)
- Gaming: Maybe (depends on game)
- Adobe Creative Suite: No (Windows/macOS only)

**Q: How do I backup my vault?**

A: 
```bash
# Option 1: Backup entire encrypted vault
dd if=/dev/mmcblk0p4 of=/backup/vault.img bs=4M

# Option 2: Backup vault contents (decrypted)
rsync -av /vault/ /backup/vault-contents/

# Option 3: Btrfs snapshots
btrfs send /vault/.snapshots/latest | ssh backup-server "btrfs receive /backups/"
```

Recommended: Option 3 (encrypted in transit, space-efficient)

**Q: Can I sync files between devices?**

A: v0.1: Manual transfer (USB, network)
v0.2: Considering encrypted sync option
Alternative: Use external sync tool in Work room (Syncthing, etc.)

**Q: What if I forget my passphrase?**

A: Data is unrecoverable (by design). 

Recommendations:
- Write passphrase on paper, store in safe
- Use passphrase manager (in vault)
- Consider key escrow service (v0.2)

No "password reset" by design (would defeat encryption).


## Getting Help

### Support Channels

**Community support (free):**
- Matrix chat: #anchor-os:matrix.org (real-time)
- Forum: https://forum.anchor-os.example (async)
- GitHub issues: Bug reports, feature requests
- Email list: users@anchor-os.example

**Documentation:**
- Official docs: https://docs.anchor-os.example
- This README and 21 other docs in `/docs`
- Community wiki: https://wiki.anchor-os.example

**Professional support (future):**
- Paid support contracts (under consideration)
- Training/consulting (TBD)
- Custom development (case-by-case)

### Before Asking for Help

**1. Check documentation**
- Read relevant section of README
- Check FAQ (above)
- Search docs folder

**2. Search existing issues**
- GitHub issues: https://github.com/anchor-os/anchor-os/issues
- Forum archives
- Matrix chat logs

**3. Gather information**
```bash
# System info
uname -a
cat /proc/version

# Bay0 logs
journalctl -u bay0 | tail -100

# Audit log (recent)
tail -50 /vault/system/audit.log

# Kernel messages
dmesg | tail -50
```

**4. Write good bug report**
- What you expected to happen
- What actually happened
- Steps to reproduce
- System info (above)
- Logs (relevant portions)

### Response Times (Expected)

**Community support:**
- Matrix chat: Minutes to hours (depends on availability)
- Forum: 1-2 days typically
- GitHub issues: 2-5 days for triage

**Security issues:**
- Initial response: 48 hours (guaranteed)
- Assessment: 1 week
- Fix: Depends on severity (critical: 48h, high: 1 week, medium: 2 weeks)

**Professional support (future):**
- TBD based on SLA


## License & Legal

### Software License

**Current:** To be determined before v0.1.0 release

**Candidates:**
- GPLv3 (strong copyleft)
- Apache 2.0 (permissive)
- MIT (permissive)
- Dual-license (commercial option)

**Criteria for decision:**
- Protect user freedom
- Enable commercial adoption
- Encourage contributions
- Prevent proprietary forks (maybe)

**Decision timeline:** Before v0.1.0 release

### Documentation License

**Proposed:** CC BY-SA 4.0 (Creative Commons Attribution-ShareAlike)

**Allows:**
- Commercial use
- Modifications
- Distribution

**Requires:**
- Attribution
- Share-alike (derivatives also CC BY-SA)

### Trademark

**"Anchor OS"** - status TBD

**Goal:** Prevent confusion, allow community use

**Expected policy:**
- Community projects: Use freely
- Commercial derivatives: Require permission
- No misleading use

### Disclaimer

```
THIS SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.

USE AT YOUR OWN RISK. THIS IS BETA SOFTWARE. DATA LOSS MAY OCCUR.

NO GUARANTEE OF SECURITY. SYSTEM MAY BE VULNERABLE TO UNKNOWN ATTACKS.

IF YOU USE THIS FOR ANYTHING IMPORTANT, YOU ACCEPT ALL RISKS.
```

**Translation:** We're honest about limitations. We try our best. But we cannot guarantee perfection. Use wisely.


## Acknowledgments

### Inspiration

**Projects that influenced Anchor OS:**
- Qubes OS (isolation model)
- ChromeOS (immutable base, verified boot)
- NixOS (declarative configuration)
- Silverblue/MicroOS (atomic updates)
- Plan 9 (clean design, everything explicit)
- seL4 (formal verification goals)

### Technologies

**Built on:**
- Linux kernel (Linus Torvalds & thousands of contributors)
- Nix package manager (Eelco Dolstra & community)
- Rust programming language (Mozilla & Rust Foundation)
- dm-verity (ChromeOS team)
- LUKS2 (cryptsetup team)
- Btrfs (Oracle & community)

### Community

**Thanks to:**
- Early testers (when available)
- Security researchers (for audits)
- Documentation contributors
- Hardware donors
- Everyone who provides feedback

**Special thanks:**
- [To be filled as project grows]


## Contact

**General inquiries:** hello@anchor-os.example

**Security issues:** security@anchor-os.example (PGP key: [link])

**Press/media:** press@anchor-os.example

**Partnerships:** partnerships@anchor-os.example

**Matrix chat:** #anchor-os:matrix.org

**GitHub:** https://github.com/anchor-os/anchor-os

**Website:** https://anchor-os.example (coming soon)

**Twitter/X:** @anchor_os (TBD)

**Mastodon:** @anchor_os@fosstodon.org (TBD)


## Final Words

Anchor OS is an experiment in honest security. We don't claim to be unhackable. We don't promise silver bullets. We don't hide limitations.

What we do promise:
- Honesty about what works and what doesn't
- Mechanical enforcement of security properties
- User ownership and control
- Simplicity over complexity
- Open development and design

If this resonates with you, join us. If not, that's okay too - use the right tool for your needs.

**The base never changes. Experiments die fast.**


**Document Version:** 1.0.0 (Extended README)  
**Last Updated:** 2026-01-21  
**Maintainer:** Anchor OS Team  
**Status:** Living document (will be updated as project evolves)

---

*This extended README is part of the complete Anchor OS documentation suite. For implementation details, see the 21 additional documents in the `/docs` folder.*

*For the complete technical handoff including all 100+ DON'Ts, failure modes, testing procedures, and kernel hardening strategies, see the main handoff document.*

**Ready to build. Ready to ship. Ready to be honest about what it is and isn't.**
