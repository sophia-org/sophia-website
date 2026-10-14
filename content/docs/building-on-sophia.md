+++
title = "Building on Sophia"
weight = 2
[extra]
version = "0.1.0"
+++

You can build a window manager, a panel, a launcher or a whole desktop on Sophia. Engine handles physical input, composition and display output. Your component works through the role files the 9P core serves to its namespace. Ordinary applications use X11 today.

Sophia is experimental, and the native application path is not open yet: the shared protocol authority contract and the application authority rebuild remain unfinished. Build against the window-manager, shell, inspection and portal contracts that run now. Check each role's version and capabilities before you depend on a feature.

## Start From Your Namespace

Your component does not open a private protocol socket. It attaches to the core, proves its identity, and receives a socket directory with one endpoint per role it may use and a discovery file that names them. The tree at each endpoint is the whole grant. The server checks attach, walk, open and every operation on an open file, so code for a tree you were not granted fails at the server, not by convention.

Small controls take a line to a `ctl` file and return state from a `status` file. Traffic on the frame path keeps compact binary records. Each file has one grammar and one version. Parse nothing partially.

A desktop bundle can ship several components together. Bundling buys shared styling and configuration. It buys no shared authority: each component still attaches with its own identity and its own tree.

## Choose What to Build

| Component | Boundary | Where it stands |
| :-- | :-- | :-- |
| Window manager | Window-manager role files: snapshots in, proposals out | Running |
| Shell or panel | Shell role files, as descriptors or as your own pixels | Running |
| Tool, status reader, agent | Inspection, output and administration files | Running in part; administration still migrates |
| Portal feature | Portal files: request, grant, bind, revoke | Portal export running; capture not yet qualified |
| Ordinary application | X11 through the X authority | Running |
| Native 9P application | Application authority export | Not yet; the contract comes first |
| New protocol frontend | A translator client of the application authority | Open to third parties once the authority lands |

## Write a Window Manager

A window manager keeps its own layout model — tiling tree, scrolling columns, workspaces — and keeps it private. Engine prescribes none of it. The exchange runs in a loop. The core serves a snapshot of opaque nodes. Your manager reads the geometries, updates its model, and writes a layout and focus proposal. Engine validates the bounds and commits geometry and pixels together. While your manager is down or restarting, Engine holds the last committed frame.

Your manager never sees titles, process IDs, clipboard contents or pixels, and it never learns which namespace a node came from. It registers named actions; the session owns shortcut matching and decides what, if anything, a launch request starts. Your manager never needs an executable path.

Start with one window, placed. Add focus and workspaces after the snapshot-and-proposal loop survives a restart of your own process.

## Write a Shell

Two models run on the shell role files. Each needs server support and an operator grant.

In the descriptor model, your shell picks from UI elements Engine knows how to draw. You receive authorised descriptors, choose order and selection, and return a candidate. Engine renders it and reports actions on the targets it presented. Switchers and launchers fit here, and your client ships no rendering stack.

In the content model, your shell draws. It rasterizes its widgets, submits the pixels with the matching interaction targets, and Engine controls placement, clipping, composition and input. Keep the phases apart in your code: allocation grants bounds, upload moves bytes into the renderer's store, presentation means the frame reached scanout, and release ends the store's custody. An accepted upload is not a presentation. Do not free a resource on close while the renderer still holds it, and apply the same care on cancel and disconnect.

Presenting a panel grants no pixels beyond your own surface. A screenshot needs a portal grant, not a read of the screen.

## Configure the Desktop

The desktop profile at `~/.config/sophia/desktop.kdl` — or `sophia/desktop.kdl` under `XDG_CONFIG_HOME` — selects components and sets their permissions. Your component keeps its own preferences: fonts, colours, layout vocabulary. The profile can carry settings for the selected window manager, and that manager validates them.

Permissions come from the operator, never from your config file. A panel may ask for forty pixels along an edge; Engine checks the ask against the profile allowance. Editing your own settings cannot widen it.

## Move Data Through Portals

Clipboard, drag-and-drop, file handoff and capture cross namespace edges, so they go through portal files. The portal export decides on bounded facts and executes nothing. Your frontend, as recipient, does the work: for a 9P client, one object binds into your tree for the life of the grant, and revocation makes an open file answer `ESTALE`. For an X client, the X authority translates the grant into X terms. A grant opens one object. It opens no namespace to inspection.

Capture is the first bind and the honest warning. The mechanism is implemented; live requests time out today, and physical qualification is unfinished. Build against the contract. Do not promise screenshots.

## Prove It

Negotiate the role version and read the result. Handle refusal as a normal outcome. Track connection epochs and record identities so a reply from a dead connection cannot move its replacement.

Test the plain exchange first, then the failures: malformed records, full queues, slow replies, disconnects, revoked grants and restarts. A refused write leaves the operation yours; a taken write leaves you tracking its outcome, not sending a second copy. Keep protocol tests off hardware. A socket test can prove ordering and ownership. It cannot prove a frame reached a panel.

## Assemble a Desktop

Pick a window manager, supply a shell, configure the session actions, ship applications. One repository or several makes no difference to the boundaries. Start with the smallest component that does one thing: a manager that places a window, a shell that presents and retires one image. Extend it after negotiation, refusal and teardown behave.
