### Cleaned & Final Version (with minimal edits)

```markdown
# Nix (Build System) — Anchor OS

This page is the authoritative description of how Anchor OS is built with Nix, what “reproducible” means here, and what is forbidden.

Anchor OS treats the build pipeline as part of the **security boundary**. The build is not “a developer convenience.” It is an **enforcement mechanism**.

---

## Principle

**Nix produces the artifact. Secure Boot decides whether it runs.**

- Nix builds the **UKI** deterministically (pinned inputs → fixed output).  
- Secure Boot verifies **signature + chain-of-trust** at boot.  
- Bay0 enforces **runtime invariants** (immutability, embedded policies, watchdog).

Any drift breaks the design.

---

## What Nix Builds

Anchor OS produces **one bootable, signed unit** per release slot (A/B):

- Hardened Linux kernel (currently aarch64-linux target)  
- Bay0 (static Rust binary, PID 1 governor)  
- Embedded room passports (policies)  
- Minimal initrd (Bay0 + runtime essentials)  
- Unified Kernel Image (**UKI**) — final EFI executable

Primary output:  
`result/anchor.efi` (unsigned) → signed as `anchor.signed.efi`

---

## Repository Layout (Expected)

```
.
├── flake.nix
├── flake.lock
├── bay0/
│   ├── Cargo.toml
│   ├── Cargo.lock
│   └── src/
├── policies/
│   └── rooms/
├── images/                     # room squashfs build recipes
│   ├── web/
│   ├── work/
│   └── vault-access/
├── kernel/
│   ├── config-hardened
│   └── patches/
├── scripts/
│   ├── build-uki.sh
│   ├── deploy-to-slot.sh
│   └── validate-kernel-config.sh
└── docs/
    ├── NIX.md                  # this page
    └── SIGNING.md
```

---

## Quick Start (Developer)

Build unsigned UKI locally:

```bash
nix build .#uki
ls -lah result/anchor.efi
```

Enter dev shell:

```bash
nix develop
```

Typical loop:

```bash
nix fmt
cargo test -p bay0
nix build .#uki
```

---

## Signing (Separated, Non-Nix)

Nix **never** touches private keys. Signing is a manual, air-gapped step.

After `nix build .#uki`:

```bash
sbsign --key PK.key --cert PK.crt \
  --output anchor.signed.efi \
  result/anchor.efi

sbverify --cert PK.crt anchor.signed.efi
```

Rules:

- CI **builds only** — never signs.  
- Signing keys **never** live on build infrastructure.  
- Release signing is a **ceremony** (see docs/SIGNING.md).

---

## Required Flake Outputs

Minimum:

```nix
packages.aarch64-linux = {
  kernel = ...;
  bay0   = ...;
  uki    = ...;
};

devShells.aarch64-linux.default = ...;
```

Recommended:

```nix
packages.aarch64-linux.roomImages = {
  web  = ...;
  work = ...;
};

checks.aarch64-linux = { ... };  # gates for CI
```

---

## Reproducibility Contract

A build is reproducible if:

1. Only pinned inputs from `flake.lock` are used.  
2. Nix sandbox is enabled (no network).  
3. UKI hash matches across two clean, controlled builders.

Anchor OS does **not** require bit-for-bit identical binaries across all hardware. It requires: same pinned inputs → same artifact on audited builders.

This enables auditability and rollback.

---

## Forbidden During Build

- Network access (`--option sandbox true` enforced)  
- `curl`, `fetchurl` with non-pinned URLs, git without commit hash  
- `sandbox = false`  
- Floating tags / channels / `latest`  
- Unpinned git refs

Enforcement: CI fails if sandbox is off or network is reachable.

---

## flake.lock Policy (Security Critical)

flake.lock updates = supply-chain event.

Rules:

- Require review by security + build owner  
- Generate dependency delta report before merge  
- Label PRs: `dependency-update`  
- No automerge bots

Workflow:

```bash
nix flake update
git diff flake.lock
nix flake metadata --json | jq .
```

---

## Kernel Build Policy

Kernel config is mechanically enforced.

Nix derivation must:

- Use `kernel/config-hardened` as baseline  
- Apply `kernel/patches/*`  
- Run validation script in `postConfigure` or `checkPhase`

Validation **must** fail the build if:

- Required hardening options missing  
- Dangerous features enabled (unprivileged BPF, io_uring, debugfs, modules, etc.)  
- Lockdown not enforced

---

## Bay0 Build Policy

Bay0 must be:

- Written in Rust  
- Memory-safe by default  
- Minimal dependencies  
- Preferably static-linked (musl target preferred)

Enforced via checks:

- LOC limit: ≤ 3000 lines in `bay0/src/` (excluding tests)  
- `unsafe` blocks: deny or require justification  
- Dependencies: cargo-deny allowlist  
- No unnecessary networking syscalls (enforced at runtime too)

Nix produces deterministic binary. CI enforces constraints.

---

## UKI Assembly

UKI embeds:

- Kernel  
- initrd (Bay0 + minimal runtime)  
- cmdline: `init=/bin/bay0 ...`  
- Embedded policies/passports (as sections or in initrd)

Nix **must** fail if:

- Policies fail schema validation  
- Required policy files missing  
- Parse errors at build time

No runtime policy fetch. No external includes.

---

## CI Gates (Minimum)

- `nix flake check`  
- `cargo test -p bay0`  
- Bay0 LOC check  
- Kernel config validation  
- Policy parse + schema check  
- QEMU smoke test (unsigned OK)  
- `cargo deny check` / dependency allowlist

CI **must not**:

- Sign artifacts  
- Deploy anything  
- Access private keys

---

## Caching (Controlled)

Binary caches allowed only if:

- Cache controlled by project  
- Artifacts signed by cache key  
- Builders trusted (or rebuilt locally for verification)

Preference: rebuild locally over untrusted cache.

---

## DevShell Minimum Contents

- Rust toolchain (rustc, cargo, rust-analyzer)  
- `sbsigntool` (test keys)  
- `efibootmgr`  
- `cryptsetup` + `tpm2-tools`  
- `qemu-system-aarch64`  
- `jq`, `tokei`, `cargo-deny`

---

## Troubleshooting

Common failures:

- Kernel config drift → fix `config-hardened` or recipe  
- Cross-compile issues → ensure musl toolchain available  
- Policy parse fail → check TOML / schema  
- UKI assembly fail → verify `ukify` inputs & paths

**Never** disable sandbox to “make it work.” That’s the wrong layer.

---

## Non-Negotiables (Summary)

- Nix builds; signing is air-gapped and separate.  
- `flake.lock` is security-sensitive.  
- Sandbox **must** be enabled.  
- No network during build.  
- Kernel config enforced mechanically.  
- Bay0 constraints enforced mechanically.

Runtime flexibility belongs in rooms — never in the build or Spine.
```

That's it.  
This version is tighter, removes a few redundancies, sharpens enforcement language, and adds the missing "Non-Negotiables" summary block for quick scanning. Everything else was already strong.

If you're committing this as `docs/NIX.md`, it's ready.
