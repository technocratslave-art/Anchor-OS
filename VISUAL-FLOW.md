
┌──────────────────────────────────────────────────────────────┐
│                         OUTSIDE ANCHOR                        │
│                                                              │
│  ┌──────────────┐                                            │
│  │   AI Flake    │                                            │
│  │ (flake.nix)   │                                            │
│  │──────────────│                                            │
│  │• nixpkgs rev │                                            │
│  │• torch + CUDA│                                            │
│  │• exact SHAs  │                                            │
│  │• train entry │                                            │
│  └──────┬───────┘                                            │
│         │ nix build / check                                  │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │ nix-room.sqsh│  (optional base plank)                     │
│  └──────┬───────┘                                            │
│         │ courier                                             │
└─────────┼────────────────────────────────────────────────────┘
          ▼

┌──────────────────────────────────────────────────────────────┐
│                           ANCHOR                              │
│                                                              │
│  ┌──────────────────────────────┐                            │
│  │        ROCK (IMMUTABLE)       │                            │
│  │──────────────────────────────│                            │
│  │• kernel                       │                            │
│  │• initramfs                    │                            │
│  │• busybox                      │                            │
│  │• room launcher                │                            │
│  │• courier                      │                            │
│  │• NO NIX, NO CUDA, NO STATE    │                            │
│  └──────────────┬───────────────┘                            │
│                 │ mounts                                     │
│                 ▼                                            │
│  ┌──────────────────────────────┐                            │
│  │            VAULT              │                            │
│  │──────────────────────────────│                            │
│  │• flakes                       │◄──── courier in/out ───┐  │
│  │• datasets                     │                         │  │
│  │• model weights                │                         │  │
│  │• room images (.sqsh)          │                         │  │
│  └──────────────┬───────────────┘                         │  │
│                 │ mounted                                  │  │
│                 ▼                                          │  │
│  ┌──────────────────────────────────────────────┐          │  │
│  │                  ROOM (PLANK)                 │          │  │
│  │──────────────────────────────────────────────│          │  │
│  │ RootFS: nix-room.sqsh                          │          │  │
│  │                                              │          │  │
│  │  /nix/store ──► tmpfs OR /vault/nix/store    │          │  │
│  │  /nix/var   ──► tmpfs OR /vault/nix/var      │          │  │
│  │                                              │          │  │
│  │  GPU: /dev/nvidia*  (passthrough)            │          │  │
│  │  Net: wan-only / restricted                  │          │  │
│  │                                              │          │  │
│  │  nix develop / nix build                     │          │  │
│  │      ▼                                       │          │  │
│  │  Model trains / runs                         │          │  │
│  │      ▼                                       │          │  │
│  │  Weights → /vault/models/                    │──────────┘  │
│  └──────────────────────────────────────────────┘             │
│                 │                                              │
│                 ▼                                              │
│          room destroyed                                        │
│          (all non-vault state gone)                             │
└──────────────────────────────────────────────────────────────┘




Replay / Portability Loop (Visual)

vault/models/my-model.gguf
        │
        ├──► courier → host
        │
        └──► courier → usb
                     │
                     ▼
               another Anchor
                     │
                     ▼
               spawn same room
               load same weights
               identical behavior




Failure Handling (Zero Drama)

room misbehaves
      │
      ▼
destroy room
      │
      ▼
fix flake
      │
      ▼
rebuild plank
      │
      ▼
same model / new outcome




Key Visual Truths (What the diagram enforces)

• Rock never touches Nix
• Vault is the only memory
• Rooms are disposable execution shells
• Flakes define reality
• Weights are the only survivors
