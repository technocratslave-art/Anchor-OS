Here’s the straight read.

1. Invariants Table
This is the constitution. It says: the base is immutable, root is read-only, no LKMs, bay0 is the only PID1, rooms are ephemeral by default, persistence only exists via explicit vault-backed mounts, cross-room flow only exists via explicit mechanisms (courier by default, shared folders only if declared), nothing ambient, network/device/syscalls are explicit and spawn-time only, updates are atomic A/B, and bay0 must stay tiny enough to audit. If any of these break, Anchor stops being Anchor.


2. Capability Freeze Rule Table
This is the gate. A feature exists in v1.5 only if it has: a named handle, always-visible on/off state, revocation cheaper than enable, bounded lifetime, and it fits flat passport declarations. If it fails any one, it’s rejected. No “mostly,” no “defer,” no exception process.


3. Passport Schema Table
This is the only policy API. Everything a room can do is declared here at spawn time: network mode, device toggles, persistence paths, shared folder declarations, clipboard + allowed channels, courier limits, and resource limits. Two system-scoped knobs exist: auto-unlock and travel mode. Anything not expressed here is denied. Nothing dynamic. No scripting.


4. Capability → Indicator Mapping Table
This table says: if power is enabled, it must be visible the entire time. Each capability has a defined indicator and location (room chrome/status bar/boot splash), and the indicator must directly expose the revoke action. No hiding indicators. No fading. No “advanced mode.”


5. Capability Lifecycle Table
This table says: every capability has a birth trigger, a deterministic death, and a one-action manual kill switch. Nothing can outlive its context. “Close room” is the universal hard reset. Anything without a full lifecycle row does not exist.



One friction point across the set (not fatal, just needs wording alignment):
Auto-unlock currently reads as “until revoked,” while the freeze rule says “no unbounded lifetimes.” If you want perfect internal consistency, either (a) explicitly allow “bounded by explicit revocation” for system-scoped capabilities, or (b) require time-bounded auto-unlock options (7/30/90 days) even if “until revoked” is also allowed.

That’s the five-table system: constitution + gate + API + visibility contract + lifecycle contract.
