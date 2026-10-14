+++
title = "Building on Sophia"
weight = 2
[extra]
version = "0.1.0"
+++

You can build a window manager, a panel, a launcher or a whole desktop on Sophia. Engine takes care of input, compositing and the display. Your component talks to Sophia through the role files the 9P core gives it. Ordinary applications use X11 for now.

Sophia is experimental, and native applications can't use it yet, because the shared contract and the application frontend aren't finished. Build against the window-manager, shell, inspection and portal roles that work today, and check each role's version and capabilities before you rely on a feature.

## Start from your namespace

Your component doesn't open a private socket with its own protocol. It attaches to the core, proves who it is, and gets a socket directory with one endpoint for each role it may use, plus a discovery file that lists them. The tree at each endpoint is everything you've been granted. The server checks every attach, walk, open, read and write, so if your code reaches for a tree it wasn't given, the server stops it. Good manners have nothing to do with it.

A small command is a line written to a `ctl` file, and state comes back from a `status` file. Frame-rate traffic uses compact binary records. Each file has one grammar and one version. If you can't parse a line completely, reject it.

You can ship several components together as a desktop bundle. They'll share styling and configuration, but not permissions: each component still attaches with its own identity and gets its own tree.

## Choose what to build

| Component | What it talks to | Status |
| :-- | :-- | :-- |
| Window manager | Window-manager role files: snapshots in, proposals out | Works today |
| Shell or panel | Shell role files, as UI elements or as your own pixels | Works today |
| Tool, status reader, agent | Inspection, output and administration files | Mostly; administration hasn't moved yet |
| Portal feature | Portal files: request, grant, bind, revoke | Works today; capture doesn't work live yet |
| Ordinary application | X11 through the X authority | Works today |
| Native 9P application | The application frontend | Not yet; the contract comes first |
| New protocol frontend | A translator that is a client of the application frontend | Open to anyone once the application frontend lands |

## Write a window manager

Your window manager keeps its own layout model, whether that's a tiling tree, scrolling columns or workspaces, and keeps it to itself. Engine doesn't impose one. The exchange is a loop. The core hands you a snapshot of anonymous nodes. You read their sizes and positions, update your model, and write back a proposal for layout and focus. Engine checks the bounds and commits geometry and pixels together. If your window manager goes down or restarts, Engine keeps the last frame on screen until it's back.

Your window manager never sees titles, process IDs, clipboard contents or pixels, and it never learns which namespace a node belongs to. It registers named actions. The session matches shortcuts to them and decides what, if anything, a launch request starts, so your window manager never needs to know a program's path.

Start small: one window, placed. Add focus and workspaces once the snapshot-and-proposal loop survives a restart of your own process.

## Write a shell

A shell can work in one of two ways. Both need server support and an operator grant.

In the first, your shell picks from UI elements Engine already knows how to draw. You receive the elements you're allowed to use, choose their order and what's selected, and send back your choice. Engine draws it and tells you when someone acts on what it showed. Switchers and launchers fit this model well, and your shell needs no rendering code of its own.

In the second, your shell draws its own widgets. It renders them, submits the pixels along with the areas that respond to input, and Engine handles placement, clipping, compositing and input. Keep four stages apart in your code. Allocation gives you a size. Upload copies your pixels into the renderer. Presentation means the frame reached the screen. Release means the renderer no longer holds your pixels. An accepted upload hasn't been shown yet. Don't free a buffer on close while the renderer still holds it, and take the same care when you cancel or disconnect.

Drawing a panel gives you no pixels outside your own surface. If you want a screenshot, ask for a portal grant; you can't just read the screen.

## Configure the desktop

The desktop profile lives at `~/.config/sophia/desktop.kdl`, or at `sophia/desktop.kdl` under `XDG_CONFIG_HOME`. It picks the components and sets their permissions. Your component keeps its own preferences, such as fonts, colors and layout options. The profile can also carry settings for the chosen window manager, and the window manager checks them itself.

Permissions come from the person running the desktop, never from your config file. A panel can ask for forty pixels along a screen edge, and Engine checks that against what the profile allows. Changing your own settings can't raise the limit.

## Move data through portals

Clipboard, drag-and-drop, file handoff and capture all cross from one namespace to another, so they go through portals. The portal service decides whether to allow a transfer and does nothing else. Your frontend, on the receiving end, does the work. A 9P client gets one object bound into its tree for as long as the grant lasts, and if the grant is revoked, the open file answers `ESTALE`. For an X client, the X authority translates the grant into X terms. A grant opens one object. It doesn't open a namespace for browsing.

Capture is the first portal, and a fair warning. It's written, but live requests time out today and it hasn't been tested on real hardware. Build against the contract, but don't promise your users screenshots yet.

## Prove it works

Negotiate the role version and read the answer. Treat a refusal as a normal result, not an error. Track connection epochs and record identities, so a late reply from a dead connection can't affect the one that replaced it.

Test the normal exchange first, then the failures: malformed records, full queues, slow replies, disconnects, revoked grants and restarts. If the server refuses a write, the operation is still yours to retry or drop. If it accepts one, don't send it again; wait for the result. Keep protocol tests off real hardware. A socket test can prove ordering and ownership, but it can't prove a frame reached a panel.

## Assemble a desktop

Pick a window manager, add a shell, set up the session's actions and ship your applications. One repository or several, the boundaries stay the same. Start with the smallest piece that does one thing: a window manager that places one window, or a shell that shows one image and takes it down again. Build on it once version negotiation, refusals and shutdown all behave.
