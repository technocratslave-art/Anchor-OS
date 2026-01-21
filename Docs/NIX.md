# Nix (Build System) — Anchor OS

This page is the authoritative description of how Anchor OS is built with Nix, what “reproducible” means here, and what is forbidden.

Anchor OS treats the build pipeline as part of the security boundary. The build is not “a developer convenience.” It is an enforcement mechanism.


## Principle

**Nix builds the artifact. Secure Boot decides whether it runs.**

- Nix produces the **UKI payload** deterministically.
- Secure Boot verifies **signature + chain-of-trust** at boot.
- Bay0 verifies **runtime invariants** (immutability, policy embedding, watchdog liveness).

If any of these drift, the design is wrong.


## What Nix Builds

Anchor OS produces **one bootable, signed unit** per release slot:

- Hardened **kernel** (AArch64)
- **Bay0** (PID 1 governor) as a static binary
- Embedded **room passports** (policies) inside the UKI
- Optional embedded initrd payload(s)
- A UKI image (EFI executable) to sign + deploy

Target output: `result/anchor.efi` (unsigned) → `anchor.signed.efi` (signed)


## Repository Layout (Expected)

. ├─ flake.nix ├─ flake.lock ├─ bay0/ │  ├─ Cargo.toml │  ├─ Cargo.lock │  └─ src/ ├─ policies/ │  └─ rooms/ ├─ images/                 # build recipes for room squashfs │  ├─ web/ │  ├─ work/ │  └─ vault-access/ ├─ kernel/ │  ├─ config-hardened │  └─ patches/ ├─ scripts/ │  ├─ build-uki.sh │  ├─ deploy-to-slot.sh │  └─ validate-kernel-config.sh └─ docs/ ├─ NIX.md               # this page └─ SIGNING.md


## Quick Start (Developer)

Build an **unsigned** UKI locally:

```bash
nix build .#uki
ls -lah result/anchor.efi

Drop into the development shell:

nix develop

Typical dev loop:

nix fmt
cargo test -p bay0
nix build .#uki


Signing (Separated, Non-Nix)

Nix never touches private signing keys. Signing is a separate operation on a separate machine.

After nix build .#uki:

sbsign --key PK.key --cert PK.crt \
  --output anchor.signed.efi \
  result/anchor.efi

sbverify --cert PK.crt anchor.signed.efi

Rules:

CI may build but must not sign.

Signing keys must never be on the build server.

Release signing is a ceremony.


(See docs/SIGNING.md.)



Flake Outputs (Required)

Your flake.nix must expose at least:

packages.aarch64-linux.kernel

packages.aarch64-linux.bay0

packages.aarch64-linux.uki

devShells.aarch64-linux.default


Optional but recommended:

packages.aarch64-linux.roomImages.web

packages.aarch64-linux.roomImages.work

checks.aarch64-linux.* for CI gates



Reproducibility Contract

A build is considered reproducible if:

1. The build uses only pinned inputs from flake.lock.


2. The build runs with Nix sandbox enabled (no network).


3. The produced UKI hash matches across two clean builders.



Anchor OS does not claim “bit-for-bit identical across all machines” as a religion. Anchor OS claims: same pinned inputs → same artifact on controlled builders.

That is what matters for auditability and rollback.



Forbidden: Network During Build

Builds must be hermetic.

No fetching from the network during nix build

No “curl in a derivation”

No “sandbox = false”

No floating inputs


Enforcement:

CI checks nix.conf (sandbox must be true)

Builds are run in environments where outbound is blocked



Forbidden: Floating Dependencies

You must not use:

latest

floating tags

unpinned git refs

unpinned nixpkgs channels

Everything is pinned by flake.lock.


flake.lock Policy (Security Critical)

flake.lock changes are treated like a supply-chain change.

Rules:

Lockfile updates require review by security + build owner

Lockfile updates require a CI job that prints dependency diffs

No “automerge dependency updates”


Recommended workflow:

nix flake update
git diff flake.lock
# produce a dependency delta report
nix flake metadata --json | jq .

Lock updates must be labeled dependency-update.



Kernel Build Policy in Nix

Kernel configuration is not “best effort.” It is a gate.

The Nix build should:

use config-hardened as the baseline

apply local patches from kernel/patches/

fail if required options drift


Example approach:

bake .config into the derivation

run a validation script in postConfigure or checkPhase


Validation must check:

modules disabled

BPF JIT disabled / unprivileged BPF disabled

io_uring disabled

debugfs disabled

lockdown enabled

required hardening flags enabled


If validation fails, the build fails.


Bay0 Build Policy in Nix

Bay0 must be:

Rust

memory-safe by default

minimal dependency set

(ideally) static-linked


Invariants to enforce via checks:

Bay0 LOC limit (≤ 3000 code lines in bay0/src/)

unsafe policy (deny or require explicit approvals)

dependency allowlist (via cargo deny)

no networking syscalls beyond AF_UNIX (runtime enforcement)


Nix’s job: produce the binary deterministically. CI’s job: enforce the constraints.



UKI Assembly

UKI must embed:

kernel

initrd (containing bay0 + minimal runtime)

cmdline specifying init=/bin/bay0

embedded policies (passports), ideally in the initrd or as UKI sections


No runtime policy fetch. No policy includes. No external policy files required at boot.

Nix must fail if:

policies fail to parse at build time

required policy files are missing

policy violates schema



CI Checks (Nix-backed)
At minimum CI must run:
1. nix flake check

2. Bay0 unit tests (cargo test)

3. Bay0 LOC limit check

4. Kernel config validation check

5. Policy parse + schema check

6. QEMU boot smoke test (unsigned is fine)

7. cargo deny / dependency allowlist checks



CI must not:

sign artifacts

deploy artifacts

handle private keys



Caching (Optional, Controlled)

Using a binary cache is allowed only if:

the cache is controlled by the project

artifacts are signed by the cache key

builders are trusted (or builds are verified by rebuilding)


If you cannot guarantee cache integrity, you rebuild locally.

Anchor OS prefers correctness over speed.



DevShell Contents (Minimum)

nix develop should include:

Rust toolchain (rustc, cargo, rust-analyzer)

sbsigntool (for local testing with test keys)

efibootmgr (for update testing)

cryptsetup + tpm2-tools (vault workflows)

qemu-system-aarch64 (integration tests)

jq, tokei, cargo-deny (CI parity)



Troubleshooting

If nix build .#uki fails:

Kernel config validation failures: fix kernel/config-hardened or the build recipe

Bay0 cross-compile failures: ensure musl target is available (or switch to glibc with justification)

Policy parse failures: fix TOML syntax or schema mismatch

UKI assembly failures: verify ukify inputs exist and paths are correct


If you’re tempted to disable sandbox to “make it work,” you’re in the wrong layer.



Non-Negotiables (Summary)

Nix builds; signing is separate.

flake.lock is security-sensitive.

sandbox must be enabled.

no network during build.

kernel config is enforced mechanically.

Bay0 constraints are enforced mechanically.


If you need runtime flexibility, you’re in the wrong layer.
