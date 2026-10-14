+++
title = "Architecture Map"
weight = 1
[extra]
version = "0.1.0"
+++

Sophia divides the desktop by who may do what. The window-manager, shell, output, lock, factotum, inspection and portal roles run as exports on the 9P core today. Administration and the broker still run on the old socket envelope. The protocol authority contract and the application authority rebuild remain open.

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
        To be named once in the protocol crate; still open.
        Authorities will feed Engine through this seam.

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
 A namespace is the tree a connection can reach. The server composes
 the tree from the admission identity and its grants, then checks
 attach, walk, open and every operation on an open file.

 ┌────────────────────────────────────┐     ┌─────────────────────────────────┐
 │ Namespace A: shared                │     │ Namespace B: confined           │
 │ X terminal | trusted tools         │     │ X browser | untrusted app       │
 │ own role trees, discovery file     │     │ own role trees, no grants       │
 └──────────────────┬─────────────────┘     └────────────────┬────────────────┘
                    │         portal grant (the only edge)   │
                    └──────────────────┬───────────────────────┘
                                       ▼
              The portal export decides. The recipient executes.
               9P recipient: bind one object into its tree.
               Revoke, and the open file answers ESTALE.
               X recipient: translate into X semantics.
               Descriptors cross on a side channel, never on 9P.
```

## Parts

**Engine** owns physical input, the scene graph, damage, frame scheduling, rendering and scanout. It speaks no client protocol. A crashed window manager leaves it holding the last committed frame on screen.

**The X authority** speaks for X11 clients. It handles X11 connections, keys every lookup by namespace, checks each request, and submits what survives as transactions. New Rust code, written from scratch. X11 is a first-class frontend, not a compatibility layer.

**The 9P core** gives every other public role one shared implementation. Each role stays an export with its own endpoint, file tree, grammar and owner. They share the core, the identity model and the descriptor rule — nothing else. Small controls take a line to a `ctl` file; state returns as a line from a `status` file. Window-manager and shell traffic keeps compact binary records. Clients find their roles in the discovery file in their socket directory.

**The window manager** holds layout and focus policy. It receives opaque nodes and returns proposals. Titles, process IDs, clipboard contents and pixels never reach it.

**Shells** draw panels, switchers and launchers under operator grants. A shell may pick from descriptors Engine renders, or submit its own pixels for an admitted surface. Presenting a panel grants no right to capture the screen.

**Portals** form the only edge between namespaces. The export decides on bounded facts and executes nothing. The recipient's frontend executes: a bind for a 9P client, a translation for an X client. Capture is the first bind — one named output, a frame record, bounded bytes over 9P, refused while locked. It is implemented, but it is not qualified yet: live requests currently time out, and physical and performance qualification remain unfinished. GPU descriptors cross on a side channel, never on the stream.

**Factotum and admission** fix identity. The first authenticated attach sets the principal and namespace for a connection. Later attaches must match it and stay inside its grants. A different identity needs a different connection. Authentication proves identity. It proves no launch custody and enlarges no grant.

**The session** composes. Recipes turn an identity and its grants into the socket directory and trees a client sees. Shared and confined profiles differ as recipes, not code paths. Observer, operator and driver name separate grant recipes. None inherits. The composed tree bounds reach inside Sophia; a containment driver bounds reach into the host, per platform.

## The Wayland question

No Wayland authority appears in the diagram because none is planned. A third party can write a Wayland compatibility layer as a 9P client of the application authority and carry its clients as sub-identities, each with its own composed tree. That path stays open. It commits Sophia to nothing.
