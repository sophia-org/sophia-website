+++
title = "Sophia by niltempus"
template = "index.html"
+++

# Sophia by niltempus

Sophia is a secure Linux desktop inspired by Plan 9, with atomic display transactions, a namespace for every client, and a native X11 server written in Rust. Window managers, panels, the screen lock and the rest of the desktop all talk to Sophia over one file protocol, 9P. Each client sees only the files it has been granted, and it proves who it is when it connects.

I started Sophia because I didn't like either of the usual choices. A traditional compositor puts layout, drawing, input routing and panels in one process, so when the window manager crashes, your session and every open application go down with it. X11 avoids that trap and falls into another: any client can read any other client's windows, clipboard and keystrokes. Sophia keeps layout policy out of the compositor, and it keeps clients from reading each other's data.

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
   ┌───────────────────────────────────────┐
   │      PROTOCOL AUTHORITY CONTRACT      │
   │   transactions in, routed input out   │
   └───────────────────┬───────────────────┘
                       ▼
   ┌───────────────────────────────────────┐
   │             SOPHIA ENGINE             │
   │ scene · commits · rendering · scanout │
   └───────────────────┬───────────────────┘
                       ▼
   ┌───────────────────────────────────────┐
   │               DRM / KMS               │
   └───────────────────────────────────────┘
```

Engine sits at the bottom. It owns the input devices, the scene graph, rendering and the display, and it doesn't speak any client protocol. It accepts transactions that have already been checked, and it sends back input, results and changes to the display layout.

Above Engine are the frontends. The X authority is Sophia's X server, which I wrote from scratch in Rust. It accepts X11 connections, keeps each client's resources inside its namespace, checks every request, and turns whatever passes into transactions for Engine. A native 9P frontend will do the same job for applications written for Sophia. Both will reach Engine through one shared contract. That contract isn't written yet.

The desktop roles sit beside them on the 9P core: window manager, shells, output, screen lock, factotum (which holds keys and proves identity), read-only inspection, and portals. Each role is a small file tree. You send a command by writing a line to its `ctl` file, and you read its state as a line from its `status` file. The window manager and shells run at frame rate, so their traffic stays in compact binary records. Two services, administration and the metadata broker, still use the old socket protocol and will move over last.

Here's what happens when a window changes. The application draws, and its frontend translates and checks the request. The window manager sees an anonymous node and proposes where it should go. Engine checks the proposal, commits the new geometry and pixels together, and flips the page, so you never see a frame where one has arrived without the other.

---

## Namespaces

A namespace is everything a connection can reach. When a client connects, it proves its identity through factotum, and that identity holds for as long as the connection lasts. The server builds the client's file tree from the identity and whatever it has been granted. The client can't build the tree or change it. Then the server checks every step: connecting, walking to a path, opening a file, and every read or write after that. Hiding a path doesn't protect anything; the server's check does.

That makes shared and confined sessions two different recipes rather than two different code paths. A trusted terminal and a confined browser get different trees. The browser can't name or read the terminal's windows, and any lookup that crosses namespaces fails.

Portals are the only way across. When a clipboard selection, a drag or a screenshot has to move between namespaces, the portal service decides whether to allow it but doesn't do the work itself. The receiving side does. A 9P client gets one object bound into its tree for as long as the grant lasts; revoke the grant, and the open file starts answering `ESTALE`. An X client gets the same thing in X terms, such as a selection or a drag. File descriptors never travel over 9P; they use a separate channel.

Screen capture is the first portal. A client asks for one output and reads back a frame record and the screenshot bytes over 9P, and the lock screen refuses the request. Capture works on both of my displays, and locking the screen or changing a display cuts off old grants as it should. Capturing mirrored displays still needs testing.

---

## X11 first, 9P native

X11 isn't a compatibility layer here. It's a full frontend with its own checks, written fresh in Rust. X clients keep their socket, DRI3 and shared memory, and none of the 9P work changes anything for them.

New software takes the 9P path. Window managers, shells, services and, later, applications written for Sophia talk to the core and get their own file trees. Agents are ordinary clients on that path: they read status files, write one-line commands and ask for portal grants like anyone else. Sophia builds nothing special for them. Observer, operator and driver are three separate sets of grants, not levels, and none of them includes another.

Wayland could run on the same contract, but it isn't on the roadmap. Someone could write a Wayland layer as a 9P client of the application frontend and carry its clients as sub-identities. I'll only write one myself if I run out of other options.

Sophia doesn't take everything from Plan 9. It has no draw protocol, because Engine already does the drawing; it doesn't treat kernel mounts as a security boundary; and its protocol replies never carry file descriptors. What it borrows is the namespace and the small file interface, laid over a compositor that commits whole frames.

---

## What you get

A confined client can't inspect another client it can't name. The clipboard, drag-and-drop and capture move between namespaces only through grants, and a grant can be revoked while it's in use. If a shell or the window manager crashes, Engine keeps the last frame on screen while the supervisor restarts it.

Every request meets the same checks, whether it arrives over X11 or 9P. Identity is fixed when the client connects, and its tree limits what it can reach. The interface is small enough that a shell script can drive the desktop by reading the discovery file and writing lines to `ctl` files.

## Status

The X authority and Engine run my desktop every day. The window-manager, shell, output, lock, factotum, inspection and portal roles all run on the 9P core, though the portals still need testing on real hardware. Administration and the metadata broker still use the old socket protocol. Authenticated connections and per-client namespaces landed on October 12 for programs the session launches. The shared contract between the frontends and Engine is still open, and so is the native application frontend. I count a piece as done when it lands and passes its tests, not before.

---
