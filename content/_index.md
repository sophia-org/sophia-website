+++
title = "Sophia Display Server"
template = "index.html"
+++

# Sophia

We have been building desktops the wrong way for thirty years.

Traditional display servers are monolithic. They force window layouts, graphics composition, input event routing, and panel rendering into a single, fragile process. If one part falters—if your status bar freezes or your tiling window manager crashes—your entire session dies. Everything you were working on vanishes.

Sophia changes this. It is a modern, transaction-driven display server and compositor that decomposes the desktop into a modular, cooperative pipeline. It reclaims the Unix philosophy: write small, specialized programs that do one thing and do it well, then coordinate them over clean, versioned boundaries.

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

Instead of one giant process holding all the cards, Sophia divides authority among four distinct, focused actors:

### 1. The Visual Kernel (Sophia Engine)
The unopinionated center of gravity. The Engine owns the physical hardware, schedules frame updates, and drives KMS/DRM presentation. It has no interest in application protocols like X11 or Wayland, nor does it care how your windows are tiled. It simply enforces visual and input authority on abstract, anonymous surfaces, committing updates atomically to prevent tearing.

### 2. The X11 Protocol Translator (X Server Frontend)
A lightweight, secure X11 translator written from scratch in Rust (`sophia-x-authority`). It listens on standard X11 sockets, manages window IDs, and translates X11 draw-calls and window allocations into clean, anonymous transactions for the Engine. It has no access to physical display devices or layout policies.

### 3. The Layout Legislator (Sophia WM)
An external process (like our reference implementation, **Hagia**) communicating over the `sophia_wm_v1` wire. It receives anonymous spatial coordinates and abstract window nodes, then returns layout and focus proposals. It operates completely blind to window titles, process IDs, and clipboard contents.

### 4. The Human Interface Coordinator (Sophia Shell)
A confined client (like our reference shell, **Narthex**) communicating over the `sophia_shell_v1` wire. The Shell specifies where panels, workspace switchers, and status bars go. It reserves edge spans for your work area and emits high-level UI descriptors, which the Engine renders securely inside a sandboxed domain.

---

## Strict Confinement with XNamespaces

Standard X11 is a security nightmare: any running application can sniff your clipboard, record your keystrokes, or inject fake inputs into other windows. There is no sandboxing.

Sophia ends this by pairing its modular architecture with **XNamespaces** and **Portals**:

*   **Isolated Namespaces:** Applications run in isolated containment domains (XNamespaces). Cross-namespace window lookups and property sharing fail closed by default. A web browser running in your untrusted namespace physically cannot see or interact with the terminal running in your secure namespace.
*   **Brokered Portals (`sophia_portal_v1`):** Clipboard sharing, drag-and-drop, and screen capture are handled by an independent, sandboxed Portal Broker. Data is never shared implicitly; transfers are treated as explicit transaction handoffs requiring your consent.

---

## True Fault Isolation

Because Sophia separates mechanism from policy, your desktop is extraordinarily resilient:

*   **Crash-Proof Sessions:** If your custom tiling window manager or panel shell crashes, your session does not go down with it. The Engine continues to run, holding your active windows in their last valid visual state on the screen, and allows your session supervisor to restart the crashed policy client instantly in the background.
*   **Write in Any Language:** You don't need to write a massive, fragile C/C++ compositor to customize your desktop. If you want a custom window layout, write a lightweight client speaking `sophia_wm_v1` in Nim, Zig, Python, or Rust. The core remains unbothered.

---
