+++
title = "Sophia by niltempus"
template = "index.html"
+++

# Sophia by niltempus

Traditional desktop compositors are monolithic. They force window layouts, graphics composition, input event routing, and panel rendering into a single process. If a tiling window manager crashes or a status bar freezes, your entire session goes down, taking your open applications with it. 

Sophia solves this by decomposing the desktop into a modular, cooperative pipeline. I have divided authority among four specialized, sandboxed programs that each do one thing well, coordinating them over clean, versioned boundaries.

---

## The Visual Pipeline

```text
                  [ sandboxed clients ]
                            │
                            ▼ (classic X11 socket)
                  ┌───────────────────┐
                  │ X SERVER FRONTEND │ ◄── [ protocol translation ]
                  └─────────┬─────────┘
                            │
                            ▼ (anonymous transactions)
                  ┌───────────────────┐
                  │   SOPHIA ENGINE   │ ◄── [ visual kernel / KMS / DRM ]
                  └──────┬─────┬──────┘
                         │     ▲
     [ sophia_wm_v1 ]    │     │      [ sophia_shell_v1 ]
    opaque layout nodes  ▼     ▼      descriptors & reservations
                  ┌───────┐   ┌───────┐
                  │ CODES │   │ CODES │ ◄── [ decoupled policy edges ]
                  │  WM   │   │ SHELL │
                  └───────┘   └───────┘
```

Sophia abandons the monolithic compositor loop. Instead, it delegates desktop authority across independent processes over versioned Unix sockets.

At the center sits `sophia-engine`. The engine owns the physical KMS/DRM frame scheduling, hardware input routing, and scene-graph composition. It knows nothing of client-facing window protocols like X11 or Wayland. It consumes only abstract, anonymous visual transaction streams.

To interface with the engine, translators like `sophia-x-authority` terminate client-side X11 connections and map application states directly to these anonymous transactions. 

Policy and chrome are similarly decoupled. External window managers negotiate geometries over the `sophia_wm_v1` protocol, operating purely on anonymous spatial nodes and focus trees. They remain completely blind to window titles, process IDs, or clipboard data. Status bars, launchers, and panel widgets coordinate screen-edge reservations over the `sophia_shell_v1` protocol.

A single window update traces this pipeline sequentially.

First, **the draw.** An application writes a draw call to its local X11 socket.

Second, **the translation.** `sophia-x-authority` intercepts the request, virtualizes the window resource, and submits an anonymous draw transaction to the engine.

Third, **the proposal.** The engine passes a spatial snapshot over `sophia_wm_v1` to the external policy supervisor. The supervisor calculates the layout and returns a geometry proposal.

Finally, **the commit.** The engine validates the geometry, synchronizes the damage regions, and page-flips the composited scene via the kernel's DRM atomic API.

No tearing. No flickering. Just clean, isolated transactions.

---

## Confinement via Namespaces

Standard X11 is a security disaster. Any running application can sniff your clipboard, record your keystrokes, or inject fake input into neighboring windows. There is no sandboxing.

Sophia fixes this by routing applications into protocol-neutral, isolated **Namespaces** mediated by secure **Portals**.

Applications run in isolated containment domains. Cross-namespace window lookups and property sharing fail closed by default. A web browser in an untrusted namespace cannot see or interact with a secure terminal session. It is physically impossible.

Common operations like clipboard sharing, drag-and-drop, and screen capture move to `sophia_portal_v1`. This sandboxed portal broker treats data transfers as explicit transactions. Nothing is shared implicitly. Every transfer requires your consent.

---

## Protocol-Neutrality: Pluggable Frontends

A display server should not be bound to the quirks of a single graphics protocol.

Traditional compositors are tightly coupled to their clients. X11 servers are hardcoded around X11 concepts; Wayland compositors are written around Wayland states. 

The `sophia-engine` is entirely protocol-neutral. It knows nothing of X11 resource trees, and it does not speak Wayland. It manages only abstract, anonymous visual transactions: buffers, damage regions, and layout epochs.

This decoupling creates a pluggable Protocol Authority Layer. 

The default translator is `sophia-x-authority`—a lightweight Rust frontend that terminates a secure subset of X11 and converts its state into engine transactions. Because the engine is agnostic, you can write and plug in any translation frontend. You could run a native Wayland frontend alongside our X11 frontend on the same display, completely side-by-side.

During early design, I drew inspiration from `XLibre`—a custom-patched, C-based Xorg server that prototypes X11 resource virtualization. `XLibre` remains our architectural blueprint for a heavyweight legacy compatibility provider, should application compatibility gaps ever justify its integration cost.

---

## Engineered System Benefits

Building a display engine from scratch around a transaction-driven architecture delivers four concrete, system-level guarantees:

First, **tear-free atomic transactions.** Window resizes, layout transitions, and pixel commits are synchronized. Tearing and flickering are prevented structurally because visual states are only page-flipped via the DRM/KMS atomic API once a layout epoch is fully settled.

Second, **modern visual rendering.** `sophia-x-authority` natively implements modern visual depth standards, exporting 24-bit TrueColor and 32-bit ARGB visuals with full alpha transparency.

Third, **hardened client confinement.** By routing applications into isolated **Namespaces**, Sophia structurally prevents cross-client pixel and input sniffing. Common vulnerabilities—such as unauthorized clipboard capturing, drag-and-drop snooping, or screen scraping—fail closed by default, requiring explicit brokered handoffs.

Fourth, **crash-proof sessions.** If your custom tiling window manager or panel shell crashes, your session does not go down. The engine continues to run, holding your active windows in their last valid visual state on the screen while your session supervisor restarts the crashed policy clients in the background.

No monolith. No single point of failure.

---
