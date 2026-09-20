+++
title = "Decomposing the Compositor"
date = 2026-09-20
[extra]
author = "niltempus"
+++

A common critique of modular desktop design goes like this: *"You claim to follow the Unix philosophy, yet Sophia contains a compositor. Isn't a compositor too complex, too unified to be a simple Unix tool?"*

It is a fair question, but it rests on a misunderstanding of what a compositor actually is—and what traditional display servers have bloated it into.

## The Monolithic Mess

Traditional compositors are monolithic. In the Wayland world, the compositor does everything. It manages display drivers, schedules page-flips, parses client protocols, calculates window layouts, draws panels, manages the clipboard, and routes input events. 

This is not a compositor; it is an operating system masquerading as a graphics loop. When your tiling window manager crashes or a panel widget freezes, your entire session dies, taking your open applications with it. This is a monolithic failure.

## The Real Job of a Compositor

At its core, compositing is not a policy job. It is a hardware-scheduling job. 

A compositor's real responsibility is simple: take pre-rendered pixel buffers, apply spatial transformations (move, scale), and schedule page-flips onto physical displays via the kernel's DRM/KMS atomic API. 

This is a graphics-hardware supervisor—a **visual kernel**. 

Like an operating system kernel, it must own the physical hardware (the screen and the input devices) to guarantee atomic composition and prevent hardware collisions. But like an operating system kernel, it should delegate policy to user space.

## Sophia's Separation of Concerns

Sophia enforces this separation of concerns by stripping all policy, chrome, and protocol interpretation out of the rendering core. 

*   **No Protocol Bloat:** `sophia-engine` does not speak X11 or Wayland. It works only with abstract, anonymous buffers and damage regions. Protocol translation is delegated to independent processes like `sophia-x-authority`.
*   **No Layout Policy:** The engine does not decide where windows go. Layout calculation is delegated to the external window manager, `Hagia`, over the `sophia_wm_v1` protocol. The window manager operates purely on anonymous spatial nodes and focus trees, remaining blind to window titles, process IDs, or application data.
*   **No Desktop Chrome:** Status bars, panels, and launchers run as independent processes, coordinating screen-edge reservations over the `sophia_shell_v1` protocol.

The engine's compositor remains "dumb." It receives pre-calculated layout transactions and page-flips them onto the screen.

## Reclaiming the Unix Philosophy

In Unix, we do not accuse the operating system kernel of violating the Unix philosophy just because it manages both memory mapping and disk writes. Those are low-level hardware enablement primitives. The kernel manages the raw hardware resources, while user-space programs handle the policy.

`sophia-engine` is the visual kernel of the desktop. It manages the display hardware and input devices, while delegating layout, presentation, and protocol parsing to specialized, independent, sandboxed programs.

By decomposing the compositor, Sophia proves that we can have near-native GPU composition and uncompromised desktop security without resorting to a fragile, monolithic loop.
