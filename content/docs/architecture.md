+++
title = "Architecture Map"
weight = 1
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

- **Sophia Engine:** The absolute visual authority. It manages physical input, visual state, frame scheduling, transaction commits, rendering, and display output.
- **Sophia X Server Frontend:** A clean, modern X11 frontend. It presents the established X11 API, translates protocol state into Sophia surface transactions, and performs X11 delivery rules. It does not control layout or scanout.
- **Sophia WM (Window Manager):** A dedicated policy process handling layout, focus, keybindings, workspaces, and launch decisions. It operates entirely on opaque layout nodes and `SurfaceId` handles.
- **Sophia Portals:** Mechanisms for deliberate cross-namespace transfers, such as clipboard sharing, drag-and-drop, and screen capture.
- **Metadata Broker and Chrome:** Translates protocol metadata into redacted compositor UI without exposing namespaces to the window manager.
- **Sophia Shell:** A separately confined client that specifies shell UI, such as panels and switchers, and requests work-area reservations. The Engine renders and presents that UI.
