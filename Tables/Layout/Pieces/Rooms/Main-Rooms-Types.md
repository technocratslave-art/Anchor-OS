Here is the **Anchor OS Rooms Table** — a clear overview of all supported room types, their purpose, default passport settings, typical use cases, and key security/usability notes.

This table is the practical "what can you actually do" guide for users (Linda, devs, tinkerers).  
Rooms are the only mutable, interactive part of the system — everything else (Spine, Bay0) is immutable.

| Room Type / Name      | Purpose / What It Does                                                                 | Default Passport Settings (key fields)                          | Typical Use Case Examples                                      | Security Notes (How It Protects)                              | Usability Notes (Linda's View)                               | Indicator Shown (Visible) |
|-----------------------|----------------------------------------------------------------------------------------|-----------------------------------------------------------------|----------------------------------------------------------------|---------------------------------------------------------------|--------------------------------------------------------------|---------------------------|
| Workshop              | Full Linux chaos sandbox — root, compilers, experiments, anything goes                | network=none, gpu=true, persist=[], clipboard=false, shared=[] | Run rootkits, test exploits, compile custom kernels, break stuff | seccomp strict (blocks dangerous syscalls), no network, tmpfs only | Devs love it — "I can do literally anything and not break my machine" | "Workshop active (full chaos)" |
| Browser               | Isolated web browsing (Firefox/Chrome) with optional persistence                      | network=wan-only, persist=["~/.mozilla"], clipboard=true (session) | Daily browsing, banking, shopping, social media                | No cross-room leak, cookies/history only in Vault, auto-clear clipboard | Feels like normal browsing, but "close tab = clean"          | "Browser active (WAN-only)" + clipboard badge |
| AI / Model Inference  | Run local LLMs (llama.cpp, Ollama, etc.) — inference or light fine-tune               | network=none, gpu=true, persist=["~/models"], clipboard=false   | Chat, code gen, image gen, local RAG, voice mode               | No network (air-gapped), weights RO from Vault, no persistence outside Vault | "I can talk to a 70B model and it forgets everything on close" | "AI active (GPU)" + "Persistent models" badge |
| Gaming / Emulator     | Run Steam, Proton, emulators (PPSSPP, Dolphin, RPCS3, etc.) with GPU passthrough      | network=none, gpu=true, persist=["~/.steam"], clipboard=false   | Gaming, retro emulation, VR if hardware supports               | No network, no mic/camera, GPU scoped, room death = no save scumming | "Play Valorant, close room, no traces left on my machine"    | "Gaming active (GPU)" |
| Workstation / Dev     | Full dev env (VS Code, IntelliJ, Android Studio, Nix devShell, etc.)                  | network=wan-only, persist=["~/projects", "~/.config"], clipboard=true | Coding, Git, testing, local dev servers                        | Scoped persistence, no host pollution, courier for git push/pull | "My projects live forever in Vault, but the workspace is always fresh" | "Dev active (WAN-only)" + "Persistent projects" |
| Android (Waydroid)    | Run full Android + apps inside Waydroid with GPU acceleration                         | network=wan-only, persist=["/data"], clipboard=false            | Mobile apps, games, testing Android features                   | Android data scoped to room, no host Android leak, room death = wipe | "Run Genshin or banking apps, close room, no Android traces" | "Android active (WAN-only)" |
| Secure / Banking      | Hardened, minimal browser room for sensitive tasks (banking, passwords, crypto)      | network=wan-only, persist=["~/.mozilla"], clipboard=false, no camera/mic | Banking, crypto wallets, 2FA, passwords                        | No mic/camera, no USB, strict seccomp, no shared folders      | "Bank, then close — no history, no cookies left behind"      | "Secure active (WAN-only)" |
| Blast Chamber         | Test untrusted models/exploits (pickle bombs, jailbreaks, rootkits)                   | network=none, gpu=true, persist=[], clipboard=false, pids_limit=512 | Test malicious models, RCE, memory bombs, jailbreak attempts   | Strict seccomp, PSI purge, no persist, no network, no clipboard | "Feed it the worst thing — watch it die in 3 seconds"        | "Blast Chamber active (GPU)" (red warning) |

### Key Notes

- **All rooms** default to **ephemeral** (tmpfs) — nothing survives unless explicitly persisted via Vault (passport + consent).
- **No room** has ambient access — everything (network, devices, persistence) is explicit and revocable.
- **Kill room** = instant purge (2–5 seconds): tmpfs unmount, cgroup removal, RAM/VRAM wipe, GPU registers reset.
- **Vault** is the only persistent store — always RO for active rooms, WO for dropzone.
- **Indicators** are mandatory — user always sees what's active (e.g. red dot for mic/camera, badge for GPU/network/persistence).

This table shows the real power:  
You can run **anything** — full OSes, VMs, malware, AI, games, browsers — but **nothing** survives unless you say so.

**“What You Can Actually Do” – Anchor Rooms Quick Reference**

This table lists every practical room type people will realistically spawn, what they’re for, default passport settings, rough resource use, and why it’s better than doing the same thing on a normal OS.

| Room Type (Common Name) | Primary Purpose / What People Use It For | Default Passport (key settings) | Typical RAM / CPU / GPU Use | Time to Spawn & Kill | Why Better Than Normal Linux / Windows / macOS | Indicator / Visible Cue |
|-------------------------|------------------------------------------|---------------------------------|-----------------------------|----------------------|------------------------------------------------|-------------------------|
| Browser Room            | Everyday web: browsing, banking, social, research | network=wan-only, persist=["~/.mozilla"], clipboard=true (session) | 2–4 GB RAM, low CPU | Spawn: ~0.5 s<br>Kill: 2 s | No history/cookies leak across sessions, no trackers survive reboot | “Browser active (WAN-only)” + clipboard badge |
| AI Inference Room       | Run local LLMs (llama.cpp, Ollama, Mistral, etc.) for chat, code, RAG | network=none, gpu=true, persist=["~/models"] | 8–32 GB RAM/VRAM, high GPU | Spawn: ~1 s<br>Kill: 3 s | No prompt leak, no cloud logging, model dies clean, weights safe in Vault | “AI active (GPU)” + “Persistent models” |
| Blast Chamber           | Test untrusted models/checkpoints/exploits (pickle bombs, jailbreaks) | network=none, gpu=true, persist=[], clipboard=false, pids_limit=512 | 16–32 GB RAM/VRAM, high GPU | Spawn: ~1 s<br>Kill: 3–5 s (purge) | Exploit dies instantly, no host leak, no persistence, perfect for red-teaming | “Blast Chamber active (GPU)” (red warning) |
| Workshop / Chaos Room   | Full root Linux sandbox: compile, hack, rootkits, experiments | network=none, gpu=true, persist=[], clipboard=false | 4–16 GB RAM, variable CPU/GPU | Spawn: ~0.5 s<br>Kill: 2 s | Can break everything safely, no host pollution, instant reset | “Workshop active (full chaos)” |
| Dev Workstation         | Coding, Git, IDEs (VS Code, IntelliJ), build tools | network=wan-only, persist=["~/projects", "~/.config"], clipboard=true | 4–12 GB RAM, medium CPU | Spawn: ~0.5 s<br>Kill: 2 s | Projects persist in Vault, workspace always clean, no global pollution | “Dev active (WAN-only)” + “Persistent projects” |
| Gaming / Emulator Room  | Steam, Proton, PPSSPP, Dolphin, RPCS3, etc. | network=none, gpu=true, persist=["~/.steam"] | 8–16 GB RAM, high GPU | Spawn: ~1 s<br>Kill: 3 s | No game saves leak, no anti-cheat detects host, instant reset | “Gaming active (GPU)” |
| Android (Waydroid)      | Mobile apps, games, testing Android features | network=wan-only, persist=["/data"], clipboard=false | 8–16 GB RAM, medium-high GPU | Spawn: ~2 s<br>Kill: 3 s | Android data scoped, no host Android leak, wipe on close | “Android active (WAN-only)” |
| Secure / Banking Room   | High-sensitivity tasks (banking, crypto, passwords) | network=wan-only, persist=["~/.mozilla"], clipboard=false, no camera/mic | 2–4 GB RAM, low CPU | Spawn: ~0.5 s<br>Kill: 2 s | No mic/camera/USB, strict seccomp, no shared folders | “Secure active (WAN-only)” (green lock icon) |

### Quick Summary for Posting / Sharing

- All rooms are **tmpfs + scoped** — nothing persists unless you explicitly say so (Vault mount).
- All rooms are **killable in 2–5 seconds** — no lingering state, no cleanup needed.
- All rooms are **passport-controlled** — network, GPU, persistence, devices — all opt-in.
- All rooms are **visible** — badges/indicators tell you what’s active right now.
- All rooms are **isolated** — compromise stays in the room, dies with the room.

This table is the “what can I actually do with this thing” cheat sheet.  
It’s the one people will screenshot and share.
