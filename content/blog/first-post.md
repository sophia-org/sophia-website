+++
title = "Introducing Sophia"
date = 2026-09-18
[extra]
author = "niltempus"
+++

I started Sophia because neither of the usual choices suited me. X11 trusts every client with every other client's windows. Wayland hands the whole desktop to one compositor and lets that process decide almost everything. I wanted the parts separated, and I wanted each part to hold only what it needs.

Sophia is a Linux desktop built that way. Engine owns the input devices and the display. The X server, the window manager, the shells and the services that move data between applications all run in their own processes and talk to Engine, or to one another, across checked boundaries. Nothing gets a say outside its own job.

## What X11 gets wrong

X11 was designed for clients that cooperate. Under the ordinary rules, a client can read another client's windows, follow its keystrokes and reach for resources that were never meant for it. A browser has no business reading a terminal, and a password prompt should not have to trust every other program on the screen, but the protocol gives them all the same room to move in.

Resize shows the same looseness. The new size and the pixels that belong at that size travel separately, so a slow client can leave the screen showing a frame built for a window that no longer has that shape. The picture tears, not because the hardware failed, but because nobody was required to finish the job before showing it.

## What Wayland gets wrong

Wayland fixed the trust problem by moving the work into the compositor. That is also its weakness. In most Wayland desktops, one process handles layout, input, panels, locks, display timing and the drawing of the screen itself. If it goes down, the session goes with it, applications and all.

The gaps in the core protocol made things worse. Clipboard, screen capture and window placement were left to extensions, and the extensions multiplied by compositor. A program that wants to do something ordinary can end up writing one backend for GNOME, another for KDE and a third for wlroots. The protocols are not identical, so the differences become the application's problem.

This is where I lost patience with it. Too many Wayland standards reach applications half-finished, or stall for years in committee and arrive divided by compositor, with basic desktop work left as an exercise for whoever writes the next extension. Nothing in that habit of mind belongs to Unix. Unix at its best gives you small programs, each doing one thing, joined by a plain interface any other program can use. Wayland gives you a committee, a monolith and a drawerful of protocols that do not quite agree.

Plan 9, and its file protocol 9P2000, appealed to me because they run the other way. One small protocol — walk, open, read, write, clunk — serves every part of the system. What a client can reach is its namespace, and the interface is files all the way down. No committee sits between the person building a part and the person using it. It is the Unix philosophy carried further than Unix itself carried it, and it is the reason Sophia took its shape from Plan 9.

## How Sophia is arranged

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

The contract in the middle is planned. The rest is how Sophia is arranged today.

Engine sits at the bottom and speaks no client protocol. It keeps the scene, takes in checked transactions, commits each window's geometry and its pixels in the same step, renders the frame and puts it on the display through DRM/KMS. If a client stalls during a resize, Engine keeps the last complete frame until the new one is ready.

Above Engine sits the X authority, a new X server written from scratch in Rust. It accepts X11 connections, keeps each client's resources inside that client's namespace and turns the requests that pass its checks into transactions for Engine. X11 is not a compatibility layer here. It is how ordinary applications run today. The server implements part of the protocol so far — DRI3, Present and MIT-SHM are among the pieces in place — and application compatibility is still in progress.

Everything else talks over one 9P2000.L core: window manager, shells, output, screen lock, factotum, read-only inspection and portals. Each role gets its own endpoint, its own file tree and its own permissions. A small command is a line written to a `ctl` file. State comes back as a line from a `status` file. The window manager and the shells move at frame rate, so their traffic uses compact binary records instead.

The window manager proposes layout and focus from anonymous nodes. It never sees a window title, a process ID or a pixel. Shells draw panels, launchers and switchers, and the window metadata they may show has been filtered by the broker first. Raw titles, process IDs and paths stay with the authority. If either one crashes, Engine keeps the last frame on screen while the session starts it again.

## Namespaces and portals

A namespace is everything a connection can reach. When a 9P client attaches, it proves who it is through factotum, and that identity holds for the life of the connection. X11 clients are admitted by the X authority on its own terms — a fresh Xauthority cookie for the session, not factotum — and then held inside the same namespace rules. The server builds the client's tree from the identity and the grants behind it, then checks every attach, walk, open, read and write. One client cannot name another client's windows, and a lookup across the boundary fails.

Portals are the only way across. The portal service decides whether a transfer may happen and does no more than decide. The receiving frontend carries it out. A 9P client gets one object bound into its tree for as long as the grant lasts; revoke the grant and the open file answers `ESTALE`. An X client gets the same transfer in X terms, as a selection. Drag-and-drop has its decision machinery, but no finished native executor yet, so I do not count it as carried. File descriptors never travel on the 9P stream. They use a separate channel.

Two pieces are still to come, and I would rather say so here than draw them as finished. The shared contract between the frontends and Engine has not been written, and neither has the native 9P frontend for applications. Administration now runs on the core by default; the old transport remains only as an explicit way back, and its installed-session acceptance is still pending. The metadata broker still uses the old socket protocol. Screen capture, the first portal, works on both of my displays; capture of mirrored displays is still being tested.

The source and the documentation are open. Read them, argue with them, and check the claims against the code.
