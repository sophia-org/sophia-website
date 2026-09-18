+++
title = "Building on Sophia"
weight = 2
+++

You can build a tiling window manager, a panel, a launcher, or a desktop
environment on Sophia. The Engine handles physical input, composition, and
display output. Your component speaks a protocol and owns a specific part of
the desktop's behavior.

Sophia is experimental. This guide explains the interfaces and their limits;
the linked protocol specifications define the messages. Check the revision and
capabilities your chosen server accepts before depending on a feature.

## Choose what to build

| What you want to build | Where it belongs | Start here |
| --- | --- | --- |
| A tiling, floating, or scrolling window manager | Spatial policy over `sophia_wm_v1` | [Hagia](https://github.com/sophia-org/hagia) |
| A switcher or shell using Engine-drawn UI | Descriptor capabilities in `sophia_shell_v1` | [Narthex](https://github.com/sophia-org/narthex) |
| A panel with its own widgets and artwork | Content capabilities in `sophia_shell_v1` | [Lom](https://github.com/sophia-org/lom) and the [content contract](https://github.com/sophia-org/sophia/blob/master/docs/content-shell.md) |
| A file manager, editor, or other application | An ordinary client of the X11 frontend | The frontend's [implementation and scope](https://github.com/sophia-org/sophia/blob/master/docs/sophia-x-authority.md) |
| A desktop environment | A chosen WM, shell components, applications, and session policy | The [desktop composition guide](https://github.com/sophia-org/sophia/blob/master/docs/desktop-composition.md) |

A desktop environment can ship these parts together, with shared themes and
configuration tools. The authority boundaries still apply: packaging a WM and
a shell in one project does not permit them to share private runtime state.

## Who controls what

The [architecture map](@/docs/architecture.md) shows the process boundaries.
Engine owns the composed scene and decides what reaches the display. The X11
frontend owns application resources and protocol rules. The WM proposes layout
and focus. Shells provide desktop UI. Session owns admission, process lifetime,
and application launch; portals authorize data transfers between namespaces.

Access to pixels is a useful test of where a feature belongs. A client can
create and read its own content. Reading another application's pixels or the
composed desktop requires authorization through Engine and the portal system.
A panel's permission to draw does not grant permission to capture the screen.

The WM has a stricter boundary. It receives opaque layout nodes and surface
handles, with no titles, process IDs, X11 resource IDs, namespace identities,
or clipboard contents. Metadata-bearing shells run in a separate protection
domain. A private WM-to-shell channel would undo that separation.

## Write a window manager

A WM is a supervised process speaking `sophia_wm_v1`. It can keep a tiling tree,
scrolling columns, floating rectangles, or some other layout model. Those are
its own data structures; Sophia does not prescribe them.

The basic exchange is:

```text
Engine snapshot -> WM policy update -> layout and focus proposal
                                      |
                                      v
                               Engine validation and commit
```

The snapshot describes the scene using opaque identities and geometry. Your
WM updates its model and submits a proposal tied to that snapshot. Engine
checks the proposal and owns the resulting visual commit. While policy is
absent or restarting, Engine preserves the last committed projection.

Start with a layout that places one window. Add focus, workspaces, and
navigation after the snapshot-and-proposal exchange works. The WM registers
semantic actions; the session and Engine own physical shortcut matching.
Launching a terminal means requesting an authorized session operation. The
WM never needs the terminal's executable path or permission to spawn it.

[Hagia](https://github.com/sophia-org/hagia) is an independent Nim implementation.
For a smaller wire example, read the
[archived revision-3 C client](https://github.com/sophia-org/sophia/tree/master/protocol/archive/sophia-wm-v1-r3).
It carries its own codec and remains unchanged so Sophia can test compatibility
with older clients. Copy it into your project if useful; keep the original
archive intact. The [WM API](https://github.com/sophia-org/sophia/blob/master/docs/sophia-wm-api.md)
and [native protocol rules](https://github.com/sophia-org/sophia/blob/master/docs/sophia-policy-ipc.md)
define negotiation, configuration activation, snapshots, proposals, and recovery.

## Write a shell

Sophia supports two shell models. Both use `sophia_shell_v1`; each capability
must be supported by the server and allowed by the operator.

### Let Engine draw the UI

A descriptor shell selects among features Engine knows how to render. It
receives authorized descriptors, chooses ordering and selection, and returns
a candidate. Engine draws the result and supplies actions for the targets it
presents.

This is the model used by [Narthex](https://github.com/sophia-org/narthex). It
suits a switcher, tabs, shortcut help, and the descriptor launcher. The feature
vocabulary limits what the shell can draw, but also reduces how much rendering
and input machinery the client must implement.

### Draw your own content

A content shell owns its widgets, typography, artwork, and layout within an
admitted surface. It rasterizes that UI and submits bounded content with the
corresponding interaction targets. Engine controls placement, clipping,
composition, and physical input.

For example, a panel can render a workspace button with its chosen font and
colors. It submits the image and the button's target together. Only after
Engine presents the matching candidate can input refer to that button. When
an action arrives, the panel updates its model and submits another candidate.

The client must distinguish several events:

- **Allocation:** the server grants space under the component's limits.
- **Upload:** immutable resource bytes reach the server's resource store.
- **Presentation:** the matching candidate reaches the display.
- **Release:** the server no longer retains the resource.

An accepted upload is not a presentation receipt. Closing a panel does not
release bytes still held by a renderer. Keep each allocation, candidate, and
resource until its own protocol outcome permits the next step; apply the same
rule during cancellation and disconnect.

[Lom](https://github.com/sophia-org/lom) develops this model with Xilem, Masonry,
and Vello. Its README distinguishes fixture results from native acceptance.
Toolkit choice does not change the wire contract. A toolkit still needs an
adapter that supplies content through Sophia's transport; selecting an
ordinary X11 or Wayland panel executable does not make it a native shell.
The [C client helpers](https://github.com/sophia-org/sophia/blob/master/bindings/c/README-shell.md)
and [shell schema](https://github.com/sophia-org/sophia/blob/master/protocol/sophia-shell-v1.kdl)
are useful starting points for an independent implementation.

Content permission grants no access to foreign pixels, arbitrary process
execution, or raw physical input. GPU execution is a separate permission.
Rendering through an admitted GPU still leaves Engine in control of
presentation. A screenshot or backdrop effect needs the relevant Engine or
portal interface, not a screen read by the shell.

## Keep configuration with its owner

The desktop profile selects components and sets their permissions. Its normal
location is `~/.config/sophia/desktop.kdl`, or `sophia/desktop.kdl` under
`XDG_CONFIG_HOME`. The [configuration guide](https://github.com/sophia-org/sophia/blob/master/docs/configuration.md)
defines the accepted syntax.

Your component keeps its own settings. A panel can choose its configuration
format, fonts, colors, and widget layout. A WM owns its layout vocabulary.
Sophia can carry WM configuration in the profile's `policy` section, but the
selected WM validates those settings.

Permission comes from the operator's profile. A panel configuration might
request forty pixels along an edge; Sophia checks that request against the
reservation allowance. Editing the panel's settings cannot enlarge that
allowance. Likewise, a launch button requests an admitted action; Session
decides which process, if any, to start.

A desktop project can provide a settings editor for these files. Preserve the
distinction between component preferences and permissions when designing it.

## Add transfers through portals

Clipboard sharing, drag-and-drop, file handoff, and screen capture cross
authority boundaries. The portal model makes the source, recipient, permitted
operation, and lifetime explicit. A successful transfer does not open a
namespace to general inspection.

Consult the [namespace and portal contract](https://github.com/sophia-org/sophia/blob/master/docs/namespaces-and-portals.md)
before building a feature that moves application data. That document separates
implemented mechanisms from planned workflows. A portal's place in the
architecture does not mean every transfer kind has a usable client API yet.

## Develop against the wire

Choose a protocol revision and negotiate capabilities. A newer server may
offer features your client does not need; an older server may omit features
your client requires. Read the negotiated result and handle refusal explicitly.
Use the connection epoch and transaction identities required by the protocol
so a reply from an old connection cannot affect its replacement.

Test ordinary exchanges first, then failure and recovery: malformed frames,
full queues, delayed replies, disconnects, revoked grants, and process restart.
A refused queue transfer must leave the unsent operation owned by the client.
Once the transport accepts it, track its outcome instead of sending a duplicate.

Sophia keeps [protocol corpora](https://github.com/sophia-org/sophia/tree/master/protocol/golden)
and independent clients alongside the implementation. Use those bytes to check
your encoder and decoder. Keep protocol tests separate from hardware tests:
a private socket test can establish message ordering and resource ownership,
but it cannot establish that a frame appeared on a physical display. The
[validation guide](https://github.com/sophia-org/sophia/blob/master/docs/validation.md)
describes the project's checks.

## Build a desktop from these parts

A desktop environment can choose a WM, supply a shell, configure session
actions, and ship ordinary applications. It can maintain those components in
one repository or several. Each still communicates through its assigned
interface and keeps its own authority.

Start with the smallest useful component: a WM that places a window, a
descriptor shell that selects an entry, or a content client that presents
and retires one image. Extend it after negotiation, refusal, and teardown
work. The [repository developer guide](https://github.com/sophia-org/sophia/blob/master/docs/building-on-sophia.md)
has the deeper contracts and design notes for each path.
