### Base Layer (/)

- `/` is a **read-only SquashFS** loop-mounted directly (`ro,loop`).  
  No tmpfs. No overlayfs. No upperdir. No writable layer of any kind.  
  Nothing in `/` is ever user-writable.

### What exists at boot (exact mounts)

- `/` → read-only SquashFS (the immutable Spine image)  
- `/dev` → devtmpfs (populated & managed by Bay0)  
- `/proc` → procfs  
- `/sys` → sysfs  
- `/boot` → empty (UKI loaded directly by EFI; never mounted as a filesystem)  
- `/proc/spine` → Bay0 runtime control/status pseudo-files  
- `/run` → tmpfs (Bay0-owned; holds sockets, logs, courier FDs, ephemeral state)  
- `/var/log/spine` → symlink → `/run/log`  
- `/persist` → bind-mounted from LUKS-encrypted Btrfs volume (Bay0-controlled; nodev,nosuid,noexec)

**Absent from base layer (does not exist at all):**

- /usr /bin /sbin /lib /etc /home /root /var (except the symlink above)  
- Nix store  
- Any package manager  
- Any shell  
- Any users  
- Any writable / executable code outside rooms

### Boot sequence

1. EFI verifies & loads UKI (kernel + initrd + Bay0 fused, signed).  
2. Kernel starts with `init=/bay0` (Bay0 is PID 1).  
3. Bay0 mounts:  
   - Spine SquashFS → `/` (ro,loop)  
   - devtmpfs → `/dev`  
   - procfs → `/proc`  
   - sysfs → `/sys`  
   - tmpfs → `/run`  
4. Bay0 exposes `/proc/spine` (simple control plane).  
5. Bay0 spawns shell-persona (Wayland compositor + UI) in isolated namespaces.

### Room launch (e.g. "open work")

1. Bay0 clones fresh mount + pid + net + ipc + user namespaces.  
2. Mounts `work.sqsh` at a private path inside the new mount namespace.  
3. pivot_root into the room's filesystem.  
4. execs room init (or minimal shell).  
5. Applies:  
   - cgroup slice (memory/CPU limits)  
   - strict seccomp filter  
   - rlimits  
   - empty capability bounding set (no CAP_SYS_ADMIN etc.)  
6. Bay0 bind-mounts only explicitly allowed subdirs from `/persist` (e.g. `/home/user/projects`), always with nodev,nosuid,noexec.  
   Rooms **cannot** execute from `/persist` or reach the vault root.

### Room exit / cleanup

- Room process tree exits → cgroup kills stragglers.  
- Mount namespace destroyed → `work.sqsh` unmounted.  
- `/bay/active` (or equivalent staging path) becomes empty again.

### Core invariants

- `/` never changes. Never writable. Never modified.  
- `/persist` is the **only** persistent writable surface — encrypted, Bay0-mediated, scoped binds only.  
- Rooms see **only** their own .sqsh + Bay0-approved binds.  
- No ambient channels, no shared mounts, no IPC between rooms except explicit courier (one-shot, mediated, logged, destroyed).  
- Reboot = full reset of all ephemeral state. Persistence survives only where explicitly configured.
