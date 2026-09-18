+++
title = "Building on Sophia"
weight = 2
+++

This is the guide for anyone who wants to build a desktop on the Sophia display server — a lean tiling window manager, a custom shell, or a full desktop environment. It defines which component owns what, which protocol each piece speaks, and how the pieces fit together.

## The One Rule

Sophia doesn't divide the desktop by feature. It divides it by who may see pixels.

- **Engine** owns the composed scene, its presentation, and access to foreign scene pixels. Clients may create and read their own content; reading another domain's pixels requires a portal grant.
- **Policy clients** — the window manager and the shell — decide what happens. They draw nothing, or they draw blind.
- **Portals** move data between confinement domains: one transfer at a time, brokered, with an identified recipient and the user's consent.

Every design question in this architecture resolves against that rule. A feature that needs to read the screen belongs to Engine, or behind a portal decision. A feature that needs application metadata is either refused or becomes a portal the application opts into. There are no exceptions for convenience, because every exception is exactly what the confinement exists to prevent.

## Visual Boundaries

Because the window manager runs out of the critical rendering loop, it cannot hang the compositor, nor does it receive security-sensitive metadata. It operates strictly on opaque node layout and focus handles.

### 1. Developing a Window Manager
A window manager client negotiates layout using the `sophia_wm_v1` protocol. It receives layout notifications when new windows are created or resized and sends transaction proposals back to the Engine.

### 2. Developing a Shell
Panels, launch menus, and notifications reside in confined shell processes. The Engine renders specific UI chrome based on high-level descriptors emitted by the shell, protecting clipboard state and user keystrokes from standard client visibility.
