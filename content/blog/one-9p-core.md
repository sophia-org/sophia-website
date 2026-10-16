+++
title = "The 9P Core"
date = 2026-10-14
[extra]
author = "niltempus"
+++

Every public part of Sophia serves from one 9P2000.L core. Each role — window manager, shells, output, screen lock, factotum, inspection, portals, administration — is an export on that core, with its own endpoint, file tree, permissions and owner. They share the server code, the identity model and one rule: file descriptors never travel on the 9P stream. They use a separate channel.

```text
  X11 clients              9P clients: WM · shells · output
        │                  lock · factotum · inspection · portals
        ▼                              │
  ┌────────────────┐          ┌────────────────────┐
  │ X AUTHORITY    │          │ ONE 9P CORE        │
  │ Native Rust    │          │ 9P2000.L           │
  │ Built from     │          │ One export         │
  │ scratch        │          │ per role           │
  └───────┬────────┘          └────────┬───────────┘
          │     admitted transactions, │
          │     proposals and commands │
          └──────────────┬─────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │          PROTOCOL AUTHORITY CONTRACT          │
 │       transactions in, routed input out       │
 └───────────────────────┬───────────────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │                SESSION OWNERS                 │
 │          admission · routing · policy         │
 └───────────────────────┬───────────────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │                 SOPHIA ENGINE                 │
 │ input · scene · commits · rendering · scanout │
 └───────────────────────┬───────────────────────┘
                         ▼
 ┌───────────────────────────────────────────────┐
 │                   DRM / KMS                   │
 └───────────────────────────────────────────────┘
```

The contract in the middle is not written yet. Today's X wiring is its first user in waiting, not the contract itself, and the native 9P application frontend waits on it too.

A small command is a line written to a `ctl` file; state is a line read from a `status` file. Window-manager and shell traffic runs at frame rate, so it keeps compact binary records. A client learns which roles it may use from a discovery file in its own socket directory. There is no registry to ask.

## Namespaces

A namespace is everything a connection can reach. A 9P client the session launches proves its identity through factotum at attach, and that identity is fixed for the connection. The server composes the client's tree from the identity and its grants, then checks every attach, walk, open, read and write. The client cannot build or alter its own tree. Hiding a path protects nothing; the server's check is the protection.

X11 clients enter through the X authority's own admission — a fresh cookie per session, not factotum — and are then held to the same rule: resources are keyed by namespace, and a lookup across the boundary fails.

```text
          9P attach                       X11 admission
   factotum + launch record          socket + session cookie
                    └────────────┬────────────┘
                                 ▼
              admission context, fixed per connection
                                 ▼
           tree composed from identity and grants
                                 ▼
   ┌───────────────────┐  portal grant  ┌───────────────────┐
   │ Namespace A       │ ─────────────▶ │ Namespace B       │
   │ shared            │  (only edge)   │ confined          │
   └───────────────────┘                └───────────────────┘

   The recipient acts: a 9P bind (revoked, the open file
   answers ESTALE) or a translation into X terms.
```

## Portals

Portals are the only way between namespaces. The portal service decides from a few fixed facts and carries nothing itself. The recipient does the work: one object bound into a 9P tree for the life of the grant, or the same transfer in X terms, such as a selection.

Screen capture is the first portal. A grant binds one named output into the requester's tree — a frame record and the screenshot bytes, up to a size limit. Locking the screen or changing a display revokes old grants. Capture works on both of my physical displays; mirrored displays are still being tested.

## Plan 9, on Linux — and, in time, FreeBSD

Plan 9 came out of Bell Labs in the late 1980s, from the people who had built Unix and wanted to push its best idea further. In Unix, many things look like files. In Plan 9, everything is a file, served over one protocol, 9P, and every process gets its own namespace — a private view of those files, assembled by binding trees together. A machine across the network could be used as naturally as a local disk, because both arrived the same way: as files in your namespace.

Sophia is not Plan 9, and it does not run on Plan 9. It runs on Linux today, with DRM/KMS displays, libinput, DMA-BUF buffers and Linux processes. What it takes from Plan 9 is the structure, not the operating system.

The borrowing is specific. The per-identity namespace becomes a tree the 9P core composes at attach and checks on every operation. Factotum, Plan 9's authentication agent, becomes the place a launched client proves who it is, so keys stay in one process. The plumber, which routed messages by rules and let the recipient do the work, becomes the portal service. Discovery by the tree replaces a registry: a client reads its discovery file and finds the roles it may use.

The departures are just as specific, and Linux is the reason for most of them. There is no draw protocol, because Engine already owns composition. Kernel mounts are never the authority — clients connect to the core directly, and the server's check, not a hidden path, decides what a client may touch. File descriptors, which no 9P reply can carry, cross on a Linux side channel, the same way DRI3 reads them at the X socket. And reach into the host — what a confined client may touch outside Sophia — belongs to a separate containment driver, not to the namespace at all.

Plan 9 also points to work Sophia has not done: an application exporting its own tree, as Acme did, or a translator re-serving window files to its own clients, as Rio did. Those wait on the application frontend. I name them here as directions, not features.

The aim is for Sophia to be cross-platform, and the Plan 9 shape is what makes that aim plausible. Only the backend, the renderer and the narrow adapter crates touch the operating system, so a port is bounded work in known places rather than a rewrite. FreeBSD is the named next port — a backend there, and a containment driver built on jail and Capsicum fed the same policy that Linux turns into bubblewrap arguments. That port is still to do: the syscall and dependency audit comes first, and I will not claim FreeBSD runs until it does. A plain 9P2000 dialect, beside today's 9P2000.L, is the later portability target for the protocol itself.

Linux supplies the hardware, the drivers and the processes today. Plan 9 supplies the shape they are arranged in — on any system Sophia learns to run on.

## Where it stands

The window-manager, shell, output, lock, factotum, inspection and portal roles run on the core today, and the X authority and Engine run my desktop every day. Administration runs on the core by default, its installed-session acceptance still pending; the metadata broker still uses the old socket protocol. The X authority implements part of X11, and compatibility work continues. Drag-and-drop has decision machinery but no finished native executor.

Agents are ordinary clients: a launched agent attaches with its own identity and tree, reads status files, writes one-line commands and asks for portal grants. Observer, operator and driver are grant recipes, not ranks, and none inherits another's.

I count a piece when it lands and passes its tests, not before.
