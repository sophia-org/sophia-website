+++
title = "Introducing Sophia: Rebuilding X11 for Security and Visual Coherence"
date = 2026-09-18
+++

We are excited to launch `sophia.gg` and share our progress on **Sophia**, a transaction-driven X11 display server and compositor designed from the ground up for strict confinement, modern visual commits, and decoupled desktop policy.

## The X11 Conundrum

For decades, X11 has been the bedrock of Unix-like desktop environments. Its flexible, asynchronous network protocol allowed cooperative clients to exchange window property lists, grab inputs, and share resources easily. However, this flexibility came with fatal flaws:

1. **No Confinement:** Any client can read the pixels or keystrokes of any other client. There is no sandboxing.
2. **Visual Tearing and Lag:** Resize operations are asynchronous. If a client lags behind the window manager, frames render with mismatched dimensions, resulting in flickering and visual garbage.

Wayland solved these problems by introducing a compositor-first model where clients render their own frames and the compositor coordinates layouts. However, Wayland pushed window layout, client coordination, and hardware driving into a single, massive monolithic compositor process. This created a new problem: if the window manager or shell crashes, your entire session and all running applications die with it.

## The Sophia Solution

Sophia takes a different path. It preserves the classic, trusted shared-X profile for applications while introducing modern visual commits and explicit confinement boundaries:

- **Transaction-Driven Rendering:** The Sophia Engine holds visual authority. Window coordinates, bounds, and surface pixels are committed *atomically*. Tearing, flicker, and black blocks are physically impossible.
- **Confronting the Shared-X Security Model:** Sophia introduces *XNamespaces*—completely isolated, virtualized X11 environments that are invisible to each other.
- **De-coupling Layout and Visuals:** Layout policy is externalized to a separate process (like our Nim reference WM, **Hagia**). The WM remains completely blind to application IDs, titles, and clipboard contents, focusing purely on opaque node operations.

By establishing absolute visual and input authority at the compositor level, Sophia demonstrates that we can have classic, modular Unix desktop architectures with the security and visual fidelity of a modern display system.

We invite you to explore our documentation, download the repository, and join us on this journey to reshape desktop computing.
