+++
title = "Sophia changed direction: one 9P core"
date = 2026-10-14
[extra]
author = "niltempus"
+++

On 9 October I decided Sophia would take its shape from Plan 9. That decision retires most of what the September posts describe. Those posts explain a desktop split across separate socket protocols, one per role, each with its own envelope and admission path. That design is going. One 9P core replaces it. Read the older posts as history.

## What changed

Every public role moves onto a single 9P server core as an export: window manager, shells, output, screen lock, factotum, read-only inspection and the portal files. The portal decision export has landed, wired to the X clipboard coordinator, and physical acceptance remains a separate test. Administration and the metadata broker still run on the old socket envelope. They migrate as exports, and the envelope retires when the last replacement passes its gate.

A namespace is now the central term. It means what a connection can reach: the role endpoints in its socket directory, and the tree each attach yields. Attach proves identity through factotum and fixes it for the life of the connection. The server derives the tree from the identity and its grants. The client does not build, store or edit it. The server checks attach, walk, open and every operation on an open file. A hidden path grants nothing. The check grants, or refuses.

Portals change with it. The portal export decides on bounded facts and executes nothing. The recipient's frontend executes. A 9P recipient receives one object bound into its tree for the life of the grant. Revocation unbinds it, and an open file answers `ESTALE`. An X recipient receives a translation into X terms: a selection, a drag, a drop. Descriptors never travel on the 9P stream. They cross on a side channel, or they stay put.

Capture goes first. One grant binds one named output into the requester's tree: a frame record and a file that returns bounded screenshot bytes. The session lock refuses capture, and locking cancels capture in flight. Bytes already delivered stay delivered. Revocation stops the next read. It cannot call pixels back. Capture is implemented, but it is not qualified yet: live requests currently time out, and physical and performance qualification remain unfinished.

## What Sophia borrows from Plan 9

Two ideas carry the load. A small uniform file interface — walk, open, read, write, clunk — serves every role. A namespace composed per identity bounds what each client can reach.

The debt does not end there. Small controls become a line written to a `ctl` file, state a line read from a `status` file. No ioctls. Frame-rate role traffic keeps compact binary records. Portal routing follows the plumber: rules over bounded facts, with the recipient doing the work. An application can export its own tree and delegate a service by the same bind that carries a portal grant, as Acme did. A translator can re-serve a window's files to its own clients, as Rio did. Factotum grows from unlocking the screen to proving identity at attach, so keys stay in one agent. The tree itself becomes the catalogue: a client reads its discovery file and finds the roles it may use. No registry.

The refusals matter as much. No draw protocol: Engine already owns composition. No kernel mounts as authority: a mount helps a client that needs paths and proves nothing about reach. No descriptors inside replies, because none crosses a mount and pure dispatch should never see one. Plan 9 supplied the structure. A transactional compositor sits in the middle.

## What did not change

Engine still owns physical input, the scene graph, visual commits, rendering and scanout, and still speaks no client protocol. The window manager still sees opaque nodes and nothing else — no titles, no process IDs, no pixels.

X11 did not shrink. It remains a first-class frontend with a native authority written from scratch in Rust. X clients keep their socket admission, namespace checks, DRI3 descriptors and shared memory. The 9P decision adds a second authority beside it, built as an export on the core once its design lands. Both authorities feed Engine through one contract, named once. The X wiring becomes the first instance of a seam, not the seam itself.

Wayland stays off the roadmap. Another developer can write a Wayland compatibility layer as a 9P client of the application authority, with its clients carried as sub-identities. I will write one myself only if the need becomes unavoidable. Possible is not planned, and the site should stop blurring the two.

Agents turn out to be ordinary clients. An agent attaches, proves an identity and receives a composed tree. It reads status files, writes one-line commands and requests portal grants. Observer, operator and driver name grant recipes, not ranks, and no recipe inherits another's grants. Sophia ships no agent harness and no feature built for one agent.

## Where this stands

The window-manager, shell, output, lock, factotum, inspection and portal roles run as exports today. The X authority and Engine run the daily session. Authenticated attach and namespace composition landed on 12 October for the admitted launch paths. Administration and the broker still migrate, the protocol authority contract and the application authority rebuild remain open, and capture still awaits qualification. Each piece counts when it lands and passes its gate.

The September posts promised pluggable frontends over socket protocols. Cross that out. The promise now reads plainer: one core, one identity proved at attach, one tree per client, and a server that checks every step.
