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

That's the difference.  
Not more features.  
Just better forgetting.
