+++
title = "Sophia changed direction: one 9P core"
date = 2026-10-14
[extra]
author = "niltempus"
+++

On October 9 I decided Sophia would take its shape from Plan 9, which retires most of what the September posts describe. Those posts explain a desktop split across separate socket protocols, one per role, each with its own envelope and admission path. One 9P core replaces that design, so read the older posts as history.

## What changed

Every public role moves onto a single 9P server: window manager, shells, output, screen lock, factotum, read-only inspection and portals. The portal service has landed and is connected to the X clipboard, though it still needs testing on real hardware. Administration and the metadata broker still use the old socket protocol. They'll move over too, and the old protocol goes away once the last of them passes its tests.

The central idea is now the namespace. It means what a connection can reach: the role endpoints in its socket directory, and the tree it gets when it attaches. A client proves its identity through factotum when it attaches, and that identity holds for the life of the connection. The server builds the tree from the identity and its grants. The client doesn't build it, store it or edit it. The server checks every attach, walk, open, read and write. Hiding a path grants nothing; only the server's check grants or refuses access.

Portals change with it. The portal service decides from a few fixed facts and carries nothing out itself. The receiving frontend does that. A 9P recipient receives one object bound into its tree for the life of the grant. Revocation unbinds it, and an open file answers `ESTALE`. An X recipient receives the same thing translated into X terms, such as a selection or a drag. Descriptors never travel on the 9P stream; they use a separate channel.

Capture goes first. One grant binds one named output into the requester's tree: a frame record and a file that returns the screenshot, up to a size limit. The session lock refuses capture, and locking cancels capture in flight. Revoking a grant stops the next read, but it can't recall bytes the client has already read. Capture now works on both physical displays in my daily session, and locking the screen or changing a display revokes old grants on time. Capturing mirrored displays, and what that costs, still needs testing.

## What Sophia borrows from Plan 9

Two ideas carry the load. A small uniform file interface — walk, open, read, write, clunk — serves every role. A namespace composed per identity bounds what each client can reach.

Sophia borrows more than that. Small controls become a line written to a `ctl` file and state a line read from a `status` file, with no ioctls. Frame-rate role traffic keeps compact binary records. Portal routing follows the plumber: rules over bounded facts, with the recipient doing the work. An application can export its own tree and delegate a service by the same bind that carries a portal grant, as Acme did. A translator can re-serve a window's files to its own clients, as Rio did. Factotum grows from unlocking the screen to proving identity whenever a client attaches, so keys stay in one place. The tree itself becomes the catalog: a client reads its discovery file to find the roles it may use, so there's no separate registry.

Sophia also leaves some things out. It has no draw protocol, because Engine already owns composition. It doesn't treat kernel mounts as a security boundary: a mount helps a client that wants paths, but it proves nothing about what the client may touch. Replies never carry file descriptors, because none can cross a mount and the code that routes messages should never handle one. Plan 9 supplies the structure, with a transactional compositor in the middle.

## What did not change

Engine still owns physical input, the scene graph, visual commits, rendering and scanout, and still speaks no client protocol. The window manager still sees only opaque nodes, never titles, process IDs or pixels.

X11 remains a first-class frontend with a native authority written from scratch in Rust. X clients keep their socket admission, namespace checks, DRI3 descriptors and shared memory. The 9P decision adds a second frontend beside it for native applications, built on the core once its design is done. Both will reach Engine through one shared contract, defined in one place. Today's X wiring becomes the first user of that contract instead of being the contract.

Wayland stays off the roadmap. Another developer can write a Wayland compatibility layer as a 9P client of the application frontend, with its clients carried as sub-identities. I'll write one myself only if I run out of other options. The September posts blurred what's possible with what's planned, and they shouldn't have.

Agents are ordinary clients. An agent attaches, proves an identity and receives a composed tree. It reads status files, writes one-line commands and requests portal grants. Observer, operator and driver name grant recipes, not ranks, and no recipe inherits another's grants. Sophia ships no agent harness and no feature built for one agent.

## Where this stands

The window-manager, shell, output, lock, factotum, inspection and portal roles run on the 9P core today, and the X authority and Engine run my desktop every day. Authenticated connections and per-client namespaces landed on October 12 for programs the session launches. Administration and the broker still have to move, the shared contract and the application frontend are still open, and mirrored capture still needs testing. I count each piece when it lands and passes its tests.

The September posts promised pluggable frontends over socket protocols. That's no longer the plan; this post describes the one that replaced it.
