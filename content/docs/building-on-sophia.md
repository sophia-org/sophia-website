+++
title = "Building on Sophia"
weight = 2
[extra]
version = "0.1.0"
+++

You can build a tiling window manager, a panel, a launcher, or an entire desktop environment on Sophia. 

The engine handles physical input, composition, and display output. Your component manages a designated desktop behavior over versioned Unix protocols.

Sophia is experimental. This guide explains the core interfaces and their constraints. Always check the negotiated protocol revision and capability flags before depending on a feature.

## Choose What to Build

| Target Component  | Subsystem Boundary                    | Reference Client                 |
| :---------------- | :------------------------------------ | :------------------------------- |
| Window Manager    | Spatial policy over `sophia_wm_v1`    | [`Hagia`][1]                     |
| Descriptor Shell  | Rendered UI over `sophia_shell_v1`    | [`Narthex`][2]                   |
| Content Panel     | Custom pixels over `sophia_shell_v1`  | [`Lom`][3]                       |
| Ordinary Client   | Application layer via X11 sockets     | [`sophia-x-authority`][4]        |
| Desktop Platform  | Unified session configuration         | [Desktop Composition Guide][5]   |

[1]: https://github.com/sophia-org/hagia
[2]: https://github.com/sophia-org/narthex
[3]: https://github.com/sophia-org/lom
[4]: https://github.com/sophia-org/sophia/blob/master/docs/sophia-x-authority.md
[5]: https://github.com/sophia-org/sophia/blob/master/docs/desktop-composition.md

A desktop environment can bundle these programs with unified styling and configuration. However, underlying authority boundaries still apply. Packaging a window manager and a shell in one release does not permit them to share private, sandboxed runtime state.

## Subsystem Authority Boundaries

The [`architecture.md`](@/docs/architecture.md) specifies process boundaries. 

The engine owns the composed scene graph and controls display scanout. The X11 frontend manages client resources and enforces protocol checks. The window manager proposes layout and focus changes. Shells supply UI elements.

Client access to pixels is strictly brokered. A client can read only its own buffers. Reading another client's pixels or capturing the composed desktop requires authorization through the engine's portal interfaces. A shell's permission to submit a status bar surface does not grant it permission to capture the screen.

The window manager operates under a strict isolation sandbox. It receives opaque layout nodes and `SurfaceId` handles. It remains completely blind to window titles, process IDs, client namespace origins, or clipboard contents.

## Write a Window Manager

A window manager runs as a supervised process communicating over `sophia_wm_v1`. It keeps its own layout models, such as tiling trees or scrolling columns. These models are entirely private. The engine does not prescribe them.

The spatial synchronization follows a strict loop.

First, **the snapshot.** `sophia-engine` transmits an anonymous scene snapshot to the window manager.

Second, **the layout.** `sophia-wm` parses the snapshot geometries and updates its layout model.

Third, **the proposal.** `sophia-wm` sends back a detailed layout and focus proposal.

Finally, **the commit.** `sophia-engine` validates the geometry bounds and commits the frame atomically.

While the policy supervisor is absent or restarting, the engine preserves the last committed projection.

Start with a layout that places one window. Add focus, workspaces, and navigation after the snapshot-and-proposal exchange works. 

The window manager registers semantic actions; the session supervisor and engine own physical shortcut matching. Launching a terminal means requesting an authorized session operation. The window manager never needs the terminal's executable path or permission to spawn it.

`Hagia` is an independent implementation written in Nim. For a minimal wire example, inspect the [archived `sophia-wm-v1-r3` C client](https://github.com/sophia-org/sophia/tree/master/protocol/archive/sophia-wm-v1-r3). It handles its own codec and remains unchanged to test backward compatibility. 

The [WM API](https://github.com/sophia-org/sophia/blob/master/docs/sophia-wm-api.md) and [native protocol rules](https://github.com/sophia-org/sophia/blob/master/docs/sophia-policy-ipc.md) define negotiation, configuration activation, snapshots, proposals, and recovery.

## Write a Shell

Sophia supports two shell models. Both use `sophia_shell_v1`. Each capability must be supported by the server and allowed by the operator.

### Let Engine Draw the UI

A descriptor shell selects among features the engine knows how to render. It receives authorized descriptors, chooses ordering and selection, and returns a candidate. The engine draws the result and supplies actions for the targets it presents.

This is the model used by `Narthex`. It suits switchers, tab layouts, and launchers. The visual vocabulary limits what the shell can draw but eliminates the need for the client to ship rendering or font-rasterization libraries.

### Draw Your Own Content

A content shell owns its widgets, typography, and layout within an admitted surface. It rasterizes its UI and submits raw pixel buffers alongside matching interaction targets. The engine controls placement, clipping, composition, and physical input.

A panel, for example, renders a workspace button with its chosen font and colors. It submits the image and the button's target bounds together. Only after the engine presents the matching candidate can input refer to that button. When an action arrives, the panel updates its model and submits another candidate.

The client must distinguish several chronological phases.

First, **allocation.** The server grants coordinate bounds under the component's configured limits.

Second, **upload.** The client transfers immutable resource bytes into the engine's GPU resource store.

Third, **presentation.** The compositor schedules the matching candidate frame and it reaches physical display scanout.

Finally, **release.** The engine destroys the resource handle and no longer retains the backing bytes.

An accepted upload is not a presentation receipt. Closing a panel does not release bytes still held by a renderer. Keep each allocation, candidate, and resource until its own protocol outcome permits the next step. Apply the same rule during cancellation and disconnect.

`Lom` develops this content model with `Xilem`, `Masonry`, and `Vello`. Its `README.md` details fixture results and native acceptance. Toolkit choice does not change the wire contract. Selecting an ordinary X11 or Wayland panel executable does not make it a native shell. 

The [C client helpers](https://github.com/sophia-org/sophia/blob/master/bindings/c/README-shell.md) and [shell schema](https://github.com/sophia-org/sophia/blob/master/protocol/sophia-shell-v1.kdl) are useful starting points for an independent implementation.

Content permission grants no access to foreign pixels, arbitrary process execution, or raw physical input. GPU execution is a separate permission. Rendering through an admitted GPU still leaves the engine in control of presentation. A screenshot or backdrop effect needs the relevant engine or portal interface, not a screen read by the shell.

## Local Configuration Management

The desktop profile selects components and sets their permissions. Its normal location is `~/.config/sophia/desktop.kdl`, or `sophia/desktop.kdl` under `XDG_CONFIG_HOME`. The [configuration guide](https://github.com/sophia-org/sophia/blob/master/docs/configuration.md) defines the accepted syntax.

Your component keeps its own settings. A panel can choose its configuration format, fonts, colors, and widget layout. A window manager owns its layout vocabulary. Sophia can carry window manager configuration in the profile's `policy` section, but the selected window manager validates those settings.

Permission comes from the operator's profile. A panel configuration might request forty pixels along an edge; the engine checks that request against the reservation allowance. Editing the panel's settings cannot enlarge that allowance. Likewise, a launch button requests an admitted action; the session manager decides which process, if any, to start.

A desktop project can provide a settings editor for these files. Preserve the distinction between component preferences and permissions when designing it.

## Add Transfers Through Portals

Clipboard sharing, drag-and-drop, file handoff, and screen capture cross authority boundaries. 

The portal model makes the source, recipient, permitted operation, and lifetime explicit. A successful transfer does not open a namespace to general inspection.

Consult the [namespace and portal contract](https://github.com/sophia-org/sophia/blob/master/docs/namespaces-and-portals.md) before building a feature that moves application data. That document separates implemented mechanisms from planned workflows. A portal's place in the architecture does not mean every transfer kind has a usable client API yet.

## Protocol-Level Validation

Choose a protocol revision and negotiate capabilities. A newer server may offer features your client does not need; an older server may omit features your client requires. Read the negotiated result and handle refusal explicitly. 

Use the connection epoch and transaction identities required by the protocol so a reply from an old connection cannot affect its replacement.

Test ordinary exchanges first, then failure and recovery. Test malformed frames, full queues, delayed replies, disconnects, revoked grants, and process restarts. A refused queue transfer must leave the unsent operation owned by the client. Once the transport accepts it, track its outcome instead of sending a duplicate.

Sophia keeps protocol corpora and independent clients alongside the implementation. Use those bytes to check your encoder and decoder. 

Keep protocol tests separate from hardware tests. A private socket test can establish message ordering and resource ownership, but it cannot establish that a frame appeared on a physical display. The [validation guide](https://github.com/sophia-org/sophia/blob/master/docs/validation.md) describes the project's checks.

## Building a Unified Desktop

A desktop environment can choose a window manager, supply a shell, configure session actions, and ship ordinary applications. It can maintain those components in one repository or several. Each still communicates through its assigned interface and keeps its own authority.

Start with the smallest useful component. Write a window manager that places a single window. Or a descriptor shell that selects an entry. Or a content client that presents and retires one image. 

Extend it only after negotiation, refusal, and teardown work.
