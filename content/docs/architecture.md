+++
title = "Architecture Map"
weight = 1
[extra]
version = "0.1.0"
+++

Sophia divides its systems based on what each part is permitted to control, rather than what is easiest to write.

```text
================================================================================
                         HARDWARE AND KERNEL
================================================================================
 [ physical input devices ]                                  [ display output ]
            │                                                        ▲
            │ raw input via libinput                                 │ DRM/KMS
            ▼                                                        │

================================================================================
                    SOPHIA ENGINE: COMPOSITOR AUTHORITY
================================================================================
 ┌────────────────────────────────────────────────────────────────────────────┐
 │ Scene graph | spatial hit-testing | damage tracking | frame scheduling     │
 │ Atomic visual commits | rendering | scanout                                │
 └───────────────┬───────────────────┬────────────────────┬───────────────────┘
          ▲      │                   │                    │      ▲
          │      │ opaque snapshots  │ portal events      │      │ descriptors & chrome
          │      ▼                   ▼                    ▼      │
 ┌───────────────┐        ┌────────────────┐       ┌─────────────────────────┐
 │  SOPHIA WM    │        │ SOPHIA PORTALS │       │      SOPHIA SHELL       │
 │ blind policy  │        │ allow/deny     │       │ panels & switchers      │
 │ layout/focus  │        │ handoff/revoke │       │ work area reservations  │
 └───────┬───────┘        └────────┬───────┘       └────────────┬────────────┘
         │                         │                            ▲
         │ layout proposals        │ portal commands            │ sanitized metadata
         │ [sophia_wm_v1]          │ [sophia_portal_v1]         │ & UI descriptors
         ▼                         ▼                            │ [sophia_shell_v1]

================================================================================
                         PROTOCOL AUTHORITY LAYER
================================================================================
 ┌────────────────────────────────────────────────────────────────────────────┐
 │ Sophia X Server Frontend: X11 resources, selections, grabs, protocol checks │
 └────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  │ namespace-checked surface transactions
                                  │ routed input / configure / lifecycle
                                  ▲

================================================================================
                         SANDBOXED CLIENT NAMESPACES
================================================================================
 ┌────────────────────────────────────┐     ┌─────────────────────────────────┐
 │ Namespace A: trusted               │     │ Namespace B: untrusted          │
 │ X terminal | trusted local tools   │  X  │ X browser | untrusted X app     │
 └────────────────────────────────────┘     └─────────────────────────────────┘
```

## System Subcomponents

To prevent privilege escalation and ensure failure isolation, the architecture enforces strict boundary checks.

**`sophia-engine`** serves as the visual kernel. It controls raw physical input, schedules frames, tracks damage, and handles scanout via DRM/KMS.

**`sophia-x-authority`** translates application protocols. It virtualizes standard X11 resources, window allocations, and input events, converting them into namespace-checked engine transactions.

**`sophia-wm`** legislates desktop policy. It calculates spatial layouts, window focus, and workspace mappings using opaque geometry node references.

**`sophia-portals`** mediates sandboxed data handoffs. It authorizes clipboard sharing, drag-and-drop actions, and display capture requests via `sophia_portal_v1`.

**`sophia-shell`** draws desktop chrome. It renders panel widgets and status indicators, reserving edge boundaries over `sophia_shell_v1`.
