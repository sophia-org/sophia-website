+++
title = "Architecture Map"
weight = 1
[extra]
version = "0.1.0"
+++

Sophia divides the desktop by who may do what. The window-manager, shell, output, lock, factotum, inspection and portal roles already run on the 9P core. Administration and the metadata broker still use the old socket protocol. Two pieces are still to come: the shared contract between the frontends and Engine, and the native application frontend.

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
 └───────────────────────────────▲───────────────────────┬────────────────────┘
                                 │                       │
              admitted transactions, proposals   routed input, outcomes,
              and metadata candidates            topology and revocation
                                 │                       │
================================================================================
                    PROTOCOL AUTHORITY CONTRACT
================================================================================
        Not written yet. Every frontend will reach
        Engine through this one contract.

================================================================================
                         PROTOCOL AUTHORITIES
================================================================================
 ┌──────────────────────────────┐   ┌─────────────────────────────────────────┐
 │ X AUTHORITY                  │   │ ONE 9P CORE (9P2000.L)                  │
 │ First-class frontend.        │   │ One export per role:                    │
 │ Native Rust, built from      │   │  window manager · shells · output       │
 │ scratch. X11 resources,      │   │  screen lock · factotum · inspection    │
 │ selections, grabs and        │   │  administration* · broker* · portal*    │
 │ protocol checks, keyed by    │   │  application authority*                 │
 │ namespace.                   │   │  (* migrating or still to land)         │
 └──────────────┬───────────────┘   └──────────────────┬──────────────────────┘
                │                                      │
       classic X11 socket                 attach: identity proved through
                │                         factotum, fixed per connection
                ▼                                      ▼

================================================================================
                         CLIENT NAMESPACES
================================================================================
 A namespace is everything a connection can reach. The server builds
 the tree from the client's proven identity and its grants, then
 checks every attach, walk, open, read and write.

 ┌────────────────────────────────────┐     ┌─────────────────────────────────┐
 │ Namespace A: shared                │     │ Namespace B: confined           │
 │ X terminal | trusted tools         │     │ X browser | untrusted app       │
 │ own role trees, discovery file     │     │ own role trees, no grants       │
 └──────────────────┬─────────────────┘     └────────────────┬────────────────┘
                    │         portal grant (the only edge)   │
                    └──────────────────┬───────────────────────┘
                                       ▼
              The portal service decides. The recipient acts.
               9P recipient: bind one object into its tree.
               Revoke, and the open file answers ESTALE.
               X recipient: translate into X semantics.
               Descriptors cross on a side channel, never on 9P.
```

## Parts

**Engine** owns the input devices, the scene graph, damage tracking, frame timing, rendering and the display. It speaks no client protocol. If the window manager crashes, Engine keeps the last committed frame on screen.

**The X authority** is Sophia's X server. It accepts X11 connections, scopes every lookup to the client's namespace, checks each request, and passes whatever survives to Engine as transactions. It's new Rust code, written from scratch. X11 is a full frontend here, not a compatibility layer.

**The 9P core** is one server implementation shared by every other public role. Each role is still its own export, with its own endpoint, file tree, grammar and owner. What they share is the core, the identity model and one rule: file descriptors never travel in the 9P stream. A small command is a line written to `ctl`, and state comes back as a line from `status`. The window manager and shells use compact binary records for their frame-rate traffic. A client finds the roles it may use in a discovery file in its socket directory.

**The window manager** decides layout and focus. It gets anonymous nodes and returns proposals. It never sees window titles, process IDs, clipboard contents or pixels.

**Shells** draw panels, switchers and launchers, with the operator's permission. A shell can choose from UI elements Engine already knows how to draw, or it can submit its own pixels for a surface it has been given. Drawing a panel doesn't give a shell the right to capture the screen.

**Portals** are the only way between namespaces. The portal service looks at a few fixed facts about a request, decides, and does nothing else. The receiving frontend carries out the decision: a bind for a 9P client, a translation for an X client. Screen capture is the first portal. It hands over one output as a frame record and screenshot bytes over 9P, and it's refused while the screen is locked. Capture works on physical displays; mirrored displays are still being tested. GPU file descriptors travel on a side channel, never in the 9P stream.

**Factotum and admission** pin down identity. The first authenticated attach on a connection sets who the client is and which namespace it gets. Any later attach has to match it and stay within the same grants, so a different identity needs a different connection. Proving who you are doesn't prove how you were launched, and it never adds to your grants.

**The session** builds each client's view. It turns an identity and its grants into the socket directory and file trees that client sees. Shared and confined profiles are different recipes, not different code. Observer, operator and driver are separate sets of grants, and none includes another. The tree limits what a client can reach inside Sophia; a containment driver, different on each platform, limits what it can reach on the host.

## Wayland

There's no Wayland frontend in the diagram because none is planned. Someone else could write one as a 9P client of the application frontend, carrying its clients as sub-identities, each with its own tree. That door stays open. Sophia isn't promising to walk through it.
