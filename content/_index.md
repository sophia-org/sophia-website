+++
title = "Sophia by niltempus"
template = "index.html"
+++

# Sophia by niltempus

Sophia takes its shape from Plan 9: one file protocol for every public role, a namespace composed for each identity, and identity proved at attach. One 9P core serves those roles. Nothing else grants reach.

The monolith desktop is a problem. A traditional compositor puts layout, composition, input routing and panels in one process. The window manager crashes, the session and its applications die with it. X11 avoids that trap and falls into another: every client can read every other client's windows, clipboard and keystrokes. Sophia separates layout policy from composition and restricts what clients can access across namespaces.

---

## The Shape

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

Engine owns physical input, the scene graph, visual commits, rendering and scanout. It knows no client protocol. It takes in admitted transactions and hands back routed input, outcomes and topology.

The X authority is a first-class frontend, written from scratch in Rust. It handles X11 connections, keys each resource by namespace, checks every request, and turns what survives into engine transactions. The 9P application authority will do the same work for native clients as an export on the core. Both will feed Engine through one contract, to be named once; that work is still open.

The desktop roles run beside them as exports on the core: window manager, shells, output, screen lock, factotum, read-only inspection and the portal files. Administration and the metadata broker still run on the old socket envelope and migrate last. A role is a file tree with a grammar. Small controls take a line written to a `ctl` file, and state returns as a line from a `status` file. Window-manager and shell traffic keeps its compact binary records. The old per-role socket protocols go away.

A window update traces a short path. The application draws. Its authority translates and checks. The window manager proposes a layout from opaque nodes. Engine validates, commits geometry and pixels together, and page-flips. No half-settled state reaches the screen.

---

## Namespaces, Composed

A namespace is what a connection can reach. Attach proves identity through factotum and fixes it for the life of the connection. The server derives the client's tree from that identity and its grants. The client never builds the tree and cannot edit it. The server checks attach, walk, open, and every operation on an open file. Hiding a path proves nothing. The check is the authority.

Shared and confined sessions become two recipes, not two code paths. A trusted terminal and a confined browser hold different trees. The browser cannot name the terminal's windows, let alone read them. Cross-namespace lookups fail closed.

Portals are the only edge between namespaces, and they work as binds. The portal export decides on bounded facts and executes nothing. The recipient's frontend does the work: a 9P recipient gets one object bound into its tree for the life of the grant, and revocation makes the open file answer `ESTALE`. An X recipient gets a translation into X semantics — a selection, a drag, a drop. Descriptors never ride the 9P stream. They cross on a side channel, or they do not cross.

The first bind is capture: one named output, a frame record, and bounded screenshot bytes read over 9P. The session lock refuses it. Capture is implemented, but it is not qualified yet: live requests currently time out, and physical and performance qualification remain unfinished.

---

## Frontends: X11 First, 9P Native

X11 is not a compatibility shim. It is a supported frontend with its own authority, admission and resource checks, in new Rust code. X clients keep their socket, DRI3 descriptors and shared memory. The 9P work changes nothing for them.

The native path is 9P. Window managers, shells, services and applications built for Sophia speak to the core and receive composed trees. An agent is an ordinary client on that path: it reads status files, writes one-line commands and asks for portal grants. Sophia builds nothing agent-specific. Observer, operator and driver are grant recipes, not ranks, and none inherits from another.

Wayland is possible through the same contract. It is not on the roadmap. Another developer can write a Wayland compatibility layer as a 9P client of the application authority and carry its clients as sub-identities. I will write one myself only if the need becomes unavoidable.

What Sophia did not take matters as much. No draw protocol. No kernel mounts as authority. No descriptors inside protocol replies. The borrow is the namespace and the small file interface, set over a transactional compositor.

---

## What This Buys

Confinement that fails closed. A confined client cannot inspect a neighbour it cannot name. Clipboard, drag-and-drop and capture cross only as grants, executed by the recipient, revoked mid-task.

Sessions that survive their parts. A shell or window manager crashes; Engine keeps the last committed frame on screen while the supervisor restarts the part.

One place to check. Attach fixes identity, the composed tree bounds reach, and every later operation meets the same checks over X11 or 9P.

A small surface to build on. Read a discovery file, open a role tree, write a line to `ctl`. A shell script can drive the desktop. So can a window manager.

## Status

The X authority and Engine run the daily session. The window-manager, shell, output, lock, factotum, inspection and portal roles serve as exports on the core; the portal files exist, and whether they pass on physical hardware is a separate question. Administration and the broker still run on the old socket envelope. Authenticated attach and namespace composition landed on 12 October for the admitted launch paths. The protocol authority contract and the application authority rebuild remain open. Capture is implemented, but it is not qualified yet. Each piece counts as shipped when it lands and passes its gate, not before.

No monolith. No ambient trust. A tree per identity, and a server that checks.

---
