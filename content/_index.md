+++
title = "Sophia Display Server"
template = "index.html"
+++

# Sophia

**Sophia** is a modern, transaction-driven display server and compositor that reclaims the **Unix Philosophy** for the graphical desktop.

Traditional compositors are monolithic blocks—violating the rule of single responsibility by forcing window layout, input event routing, hardware graphics composition, and panel UI rendering into a single, crash-prone process.

Sophia decomposes the modern desktop into a modular **visual pipeline** of specialized, single-responsibility processes, communicating over elegant, versioned IPC interfaces.

---

## The Visual Pipeline

```text
               [ physical hardware / KMS / DRM ]
                              │
                              ▼
                    ┌───────────────────┐
                    │   SOPHIA ENGINE   │ ◄── [ mechanism / visual authority ]
                    └──────┬─────┬──────┘
                           │     ▲
       [ sophia_wm_v1 ]    │     │      [ sophia_shell_v1 ]
      opaque layout nodes  ▼     ▼      descriptors & reservations
                    ┌───────┐   ┌───────┐
                    │ CODES │   │ CODES │ ◄── [ decoupled policy edges ]
                    │  WM   │   │ SHELL │
                    └───────┘   └───────┘
```

Instead of a singular monolithic process, Sophia divides authority among independent actors that each do one thing, and do it well:

### 1. The Visual Kernel (Sophia Engine)
The unopinionated core. It manages physical hardware, raw libinput events, atomic visual commits, and DRM/KMS presentation. It does not understand application protocols like X11 or Wayland, nor does it know how windows should be laid out. It enforces visual and input authority on abstract, anonymous surfaces.

### 2. The X11 Protocol Translator (X Server Frontend)
A clean, modern, secure X11 subset translator written entirely from scratch in Rust (`sophia-x-authority`). It listens on classic X11 sockets, manages resource IDs (XIDs) and selections, and translates X11 draw-calls and window allocations into anonymous transaction facts for the Engine. It has no access to physical devices or final layouts.

### 3. The Layout Legislator (Sophia WM)
An external process (like our Nim reference WM, **Hagia**) communicating over the `sophia_wm_v1` wire. It receives anonymous spatial coordinates and window handles, and outputs layout and focus proposals. It is completely blind to window titles, process IDs, or clipboard content, calculating layouts on abstract nodes.

### 4. The Human Interface Coordinator (Sophia Shell)
A confined shell client (like our reference shell, **Narthex**) communicating over the `sophia_shell_v1` wire. It specifies panels, status indicators, and workspaces. It reserves edge spans for work areas and emits high-level UI descriptors which are rendered securely inside a sandboxed domain by the Engine.

---

## Decoupled Architecture, True Fault Isolation

Because Sophia separates mechanism from policy, the desktop achieves absolute resilience:

*   **Robustness:** If your custom tiling window manager or panel shell crashes, **your session does not die**. The Engine continues to run, holding the last valid visual state on screen, and allows the session supervisor to restart the crashed clients instantly in the background.
*   **Hackability:** You can write a window manager or a status panel in any language (Rust, Nim, Zig, or Python) simply by implementing the unopinionated `sophia_wm_v1` or `sophia_shell_v1` protocols.

---
