+++
title = "Introducing Sophia: A Secure, Transaction-Driven X11 Display Server"
date = 2026-09-18
+++

We are launching `sophia.gg` to document our progress on **Sophia**, an experimental, transaction-driven X11 display server and compositor. It introduces strict client confinement, synchronized visual commits, and decoupled desktop policy.

## The Limits of X11

The X11 protocol was designed for cooperation, not security. Its flexible, asynchronous design allows clients to share window properties, grab inputs, and read each other’s resources. But this trust creates two significant liabilities:

1. **No Isolation:** Any client can read the pixels or keystrokes of another client. A compromised browser can easily inspect your terminal session or password prompt.
2. **Asynchronous Visual Tearing:** Window resize operations are decoupled from pixel rendering. If an application lags behind the window manager, frames render with mismatched dimensions, causing flickering and transient visual artifacts.

Wayland addressed these issues by isolating client pixels and forcing client-side rendering. However, it also shifted window layout, client coordination, and hardware display drivers into a single, monolithic compositor. If a Wayland compositor crashes, your entire session dies, and all active applications are lost.

## The Sophia Architecture

Sophia takes a different approach. It preserves a classic, trusted X11 environment for cooperative applications while introducing strict visual transactions and explicit confinement boundaries:

*   **Transaction-Driven Rendering:** The Sophia Engine acts as the visual authority. It commits window geometry and surface pixels atomically. If a client lags during a resize, the compositor retains the last valid visual state until the transaction is ready, preventing visual garbage and flickering by design.
*   **XNamespaces:** Instead of a single shared environment, Sophia introduces virtualized, isolated X11 domains. Cross-namespace lookups fail closed by default. A web browser running in one namespace cannot inspect or interact with a terminal running in another.
*   **Decoupled Layout and Visuals:** Layout policy is externalized to an independent process, such as our reference window manager, **Hagia**. The window manager operates purely on opaque layout nodes, remaining blind to window titles, client process IDs, and clipboard contents.

By separating protocol translation, layout policy, and visual composition into independent processes, Sophia demonstrates that we can retain modular Unix desktop architectures without sacrificing security or visual coherence.

We invite you to explore our documentation, review the source code, and follow our development as we refine the implementation.
