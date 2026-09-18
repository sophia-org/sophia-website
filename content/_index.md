+++
title = "Sophia Display Server"
template = "index.html"
+++

# Sophia

**Sophia** is a modern, transaction-driven display server and compositor that applies the **Unix Philosophy** to the graphical desktop.

Traditional compositors are monolithic. They force window layouts, graphics composition, input event routing, and panel rendering into a single, complex process. If one part falters—if your status bar freezes or your tiling window manager crashes—your entire session goes down with it, taking your open applications with it.

Sophia changes this by decomposing the desktop into a modular, cooperative pipeline. It divides authority among small, specialized programs that each do one thing well, coordinating them over clean, versioned boundaries.

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

## Protocol-Neutrality: Pluggable Frontends

A display server should not be bound to the quirks of a single graphics protocol. 

Traditional compositors are tightly coupled to their client protocols—X11 servers are hardcoded around X11 concepts, and Wayland compositors are written around Wayland-specific surface states. 

Sophia is different. The Engine is entirely protocol-neutral. It has no understanding of X11 resource trees, nor does it speak Wayland. It manages only abstract, anonymous visual transactions (buffers, damages, and layout epochs). 

This decoupling establishes a highly modular Protocol Authority Layer:

*   **The Default Rust Frontend (`sophia-x-authority`):** Our lightweight translator that terminates a secure, modern subset of X11 and converts its state into Engine transactions.
*   **Pluggable Adapters:** Because the Engine boundary is agnostic, any developer can write a translation frontend. You could plug in a native Wayland translator, support a future custom protocol, or even run multiple frontend translators simultaneously on the same visual canvas.
*   **The Legacy Seam (XLibre):** During early design, we drew inspiration from XLibre—an archived, custom-patched C-based Xorg server that prototyped X11 resource virtualization and routed inputs. While retained as historical evidence outside our production code, XLibre serves as an architectural blueprint for a heavyweight compatibility provider, should legacy application gaps ever justify its maintenance cost.

---

## Engineered System Benefits

By rewriting the display stack from scratch in Rust and adopting a compositor-first architecture, Sophia delivers concrete system-level guarantees that traditional display servers cannot match:

*   **Tear-Free Atomic Transactions:** The Sophia Engine holds visual authority. Window resizes, layout transitions, and pixel commits are synchronized. Tearing and flickering are prevented structurally because visual states are only page-flipped via the DRM/KMS atomic API once a layout epoch is fully settled.
*   **TrueColor Depth (24-bit & 32-bit):** The X Server Frontend natively implements modern visual depth standards. It exports 24-bit TrueColor and 32-bit ARGB visuals (with alpha channel support), guaranteeing sharp text rendering and native alpha transparency.
*   **Crash-Proof Sessions:** If your custom tiling window manager or panel shell crashes, your session does not go down. The Engine continues to run, holding your active windows in their last valid visual state on the screen, while your session supervisor restarts the crashed policy clients instantly in the background.
*   **Language-Neutral Extension:** You don't need to write a massive, fragile C/C++ compositor to customize your desktop. You can write a lightweight tiling window manager speaking `sophia_wm_v1` or a custom panel speaking `sophia_shell_v1` in Nim, Zig, Python, or Rust. The core visual kernel remains unbothered.

---
