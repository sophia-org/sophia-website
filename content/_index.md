+++
title = "Sophia by niltempus"
template = "index.html"
+++

# Sophia by niltempus

Sophia is a desktop built on an idea from Plan 9: every client gets its own view of the system, and the server checks every step it takes. Window managers, panels, the screen lock and the rest of the desktop all talk to Sophia over one file protocol, 9P. Each client sees only the files it has been granted, and it proves who it is when it connects.

I started Sophia because I didn't like either of the usual choices. A traditional compositor puts layout, drawing, input routing and panels in one process, so when the window manager crashes, your session and every open application go down with it. X11 avoids that trap and falls into another: any client can read any other client's windows, clipboard and keystrokes. Sophia keeps layout policy out of the compositor, and it keeps clients out of each other's business.

---

## The shape

```text
  X11 clients            9P clients: window manager · shells · output
        │                lock · administration · inspection · agents
        ▼                         │
  ┌───────────────┐         ┌────────────────┐
  │  X AUTHORITY  │         │  ONE 9P CORE   │
  │ native Rust,  │         │  9P2000.L      │
  │ built from    │         │  one export    │
  │ scratch       │         │  per role      │
  └───────┬───────┘         └───────┬────────┘
          └────────────┬────────────┘
                       ▼
        PROTOCOL AUTHORITY CONTRACT
        transactions in, routed input out
                       ▼
               SOPHIA ENGINE
        scene · commits · rendering · scanout
                       ▼
                   DRM / KMS
```

Engine sits at the bottom. It owns the input devices, the scene graph, rendering and the display, and it doesn't speak any client protocol. It accepts transactions that have already been checked, and it sends back input, results and changes to the display layout.

Above Engine are the frontends. The X authority is Sophia's X server, which I wrote from scratch in Rust. It accepts X11 connections, keeps each client's resources inside its namespace, checks every request, and turns whatever passes into transactions for Engine. A native 9P frontend will do the same job for applications written for Sophia. Both will reach Engine through one shared contract. That contract isn't written yet.

The desktop roles sit beside them on the 9P core: window manager, shells, output, screen lock, factotum (which holds keys and proves identity), read-only inspection, and portals. Each role is a small file tree. You send a command by writing a line to its `ctl` file, and you read its state as a line from its `status` file. The window manager and shells run at frame rate, so their traffic stays in compact binary records. Two services, administration and the metadata broker, still use the old socket protocol and will move over last.

Here's what happens when a window changes. The application draws. Its frontend translates the request and checks it. The window manager sees an anonymous node and proposes where it should go. Engine checks the proposal, commits the new geometry and the new pixels together, and flips the page. You never see a frame where one has arrived without the other.

---

## Namespaces

A namespace is everything a connection can reach. When a client connects, it proves its identity through factotum, and that identity holds for as long as the connection lasts. The server builds the client's file tree from the identity and whatever it has been granted. The client can't build the tree or change it. Then the server checks every step: connecting, walking to a path, opening a file, and every read or write after that. Hiding a path isn't security. The check is.

That makes shared and confined sessions two different recipes rather than two different code paths. A trusted terminal and a confined browser get different trees. The browser can't even name the terminal's windows, let alone read them, and any lookup that crosses namespaces fails.

Portals are the only way across. When a clipboard selection, a drag or a screenshot has to move between namespaces, the portal service decides whether to allow it but doesn't do the work itself. The receiving side does. A 9P client gets one object bound into its tree for as long as the grant lasts; revoke the grant, and the open file starts answering `ESTALE`. An X client gets the same thing in X terms: a selection, a drag, a drop. File descriptors never travel over 9P. They take a side channel, or they don't move at all.

Screen capture is the first portal. A client asks for one output and reads back a frame record and the screenshot bytes over 9P, and the lock screen refuses the request. Capture is written, but it doesn't work live yet: requests time out, and it hasn't been tested on real hardware.

---

## X11 first, 9P native

X11 isn't a compatibility layer here. It's a full frontend with its own checks, written fresh in Rust. X clients keep their socket, DRI3 and shared memory, and none of the 9P work changes anything for them.

New software takes the 9P path. Window managers, shells, services and, later, applications written for Sophia talk to the core and get their own file trees. Agents are ordinary clients on that path: they read status files, write one-line commands and ask for portal grants like anyone else. Sophia builds nothing special for them. Observer, operator and driver are three separate sets of grants, not levels, and none of them includes another.

Wayland could run on the same contract, but it isn't on the roadmap. Someone could write a Wayland layer as a 9P client of the application frontend and carry its clients as sub-identities. I'll only write one myself if I run out of other options.

What Sophia leaves out of Plan 9 matters as much as what it takes. There's no draw protocol, because Engine already does the drawing. Kernel mounts aren't a security boundary. Protocol replies never carry file descriptors. What Sophia borrows is the namespace and the small file interface, laid over a compositor that commits whole frames.

---

## What you get

Confinement that fails safe. A confined client can't inspect a neighbor it can't name. The clipboard, drag-and-drop and capture cross only through grants, and a grant can be pulled mid-task.

A session that outlives its parts. If a shell or the window manager crashes, Engine keeps the last frame on screen while the supervisor restarts it.

One set of checks. Identity is fixed when a client connects, its tree limits what it can reach, and every request after that meets the same checks, whether it arrives over X11 or 9P.

Something small to build on. Read a discovery file, open a role's tree, write a line to `ctl`. A shell script can drive the desktop. So can a window manager.

## Status

The X authority and Engine run my desktop every day. The window-manager, shell, output, lock, factotum, inspection and portal roles all run on the 9P core, though the portals still need testing on real hardware. Administration and the metadata broker still use the old socket protocol. Authenticated connections and per-client namespaces landed on October 12 for programs the session launches. The shared contract between the frontends and Engine is still open, and so is the native application frontend. I count a piece as done when it lands and passes its tests, not before.

No monolith, no ambient trust: one tree per identity, and a server that checks every step.

---
