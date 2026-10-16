+++
title = "What 'Compositing' means in the Sophia Engine"
date = 2026-09-20
[extra]
author = "niltempus"
+++

A fair objection to Sophia goes like this: you praise small, separate programs, and then you build a compositor. Is a compositor not the opposite of that — one large program doing everything at once?

It would be, if compositing meant everything. It does not. The word has collected jobs the way a junk drawer collects batteries, and most of the trouble people blame on compositors belongs to the extra jobs, not to the compositing.

## The job itself

Compositing, stripped down, is a scheduling job with a screen attached. Take buffers that are already drawn, put each one where it belongs, and hand the finished frame to the display through the kernel's DRM/KMS interface at the moment the hardware can take it. That is all.

Somebody has to own the display and the input devices while doing it. Two programs cannot page-flip the same screen on their own schedules, and a keystroke has to go to one window, not to whichever program asks last. Engine owns the hardware for the same reason a kernel owns a disk: not to make policy, but to keep the claimants from colliding.

## What grew around it

Traditional servers did not stop there. An X server grew protocol interpretation, resource tables and input policy around the drawing. A Wayland compositor grew layout, panels, locks and timing around it. The result is one process whose crash takes the session, the applications and the screen down together, and whose inner parts cannot be replaced one at a time.

People then argue about compositors when they mean monoliths. I would rather argue about the monolith.

## What Sophia leaves out

Engine keeps the small job and gives the rest away.

It speaks no client protocol. The X authority, a separate process written from scratch in Rust, parses X11, keeps each client's resources in its namespace and sends Engine checked transactions. The desktop roles — window manager, shells, output, lock, factotum, inspection and portals — reach Sophia through one shared 9P2000.L core, each with its own file tree and permissions.

It makes no layout policy. The window manager receives anonymous nodes, works out where things should go and sends back a proposal. Engine checks the proposal and commits it. The manager never sees a title, a process ID or a pixel, and it can be rewritten, or restarted, without Engine learning anything new.

It draws no panels. A shell either chooses among elements Engine knows how to draw or submits its own pixels for a surface it has been given. Engine places the result and routes the clicks. Owning a panel earns a shell no view of anyone else's windows; a screenshot still takes a portal grant.

It moves no data between clients. Portals decide whether a clipboard selection or a capture may cross from one namespace to another, and the receiving frontend does the carrying. Drag-and-drop decisions exist; the native carrying does not yet.

What is left in Engine is the frame: the scene, the atomic commit of geometry and pixels together, rendering and scanout. If the window manager or a shell dies, Engine holds the last complete frame on screen until the session brings the program back.

## The kernel comparison, kept honest

Calling Engine a visual kernel is an analogy, and analogies earn their keep only if you state where they stop. Engine is not a kernel. It does not schedule your programs or manage your memory. The comparison is about the division of labour. A kernel owns the hardware and refuses, on the whole, to care what the programs above it are for. Engine owns the screen and the input devices and refuses to care which protocol a window arrived by or what its title says.

Write the compositor that way and it stops being a monolith with better marketing. It becomes the piece at the bottom that everything else can afford to lose, restart or replace — because the frame stays up while they do it.
