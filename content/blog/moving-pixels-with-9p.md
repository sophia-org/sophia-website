+++
title = "Moving Pixels with 9P2000.L"
date = 2026-10-18
[extra]
author = "niltempus"
+++

9P is good at almost everything Sophia asks of it. It carries files, names, permissions, control messages and events. It is how the window manager, the shells, the screen lock and the other parts of the system talk to the session.

Pixels are the one hard case.

## Sharing pixels through a handle

On Linux, programs can share graphics storage through file descriptors. A producer fills a buffer and passes a descriptor to the compositor, which can map the same backing. This avoids sending the pixel payload through the control connection. Preparing the buffer and rendering it can still require copies; sharing a descriptor does not establish an end-to-end zero-copy path.

9P cannot hand over a descriptor. It only moves bytes.

There are good reasons for this. 9P was made to work the same over a cable, a pipe or a socket, on any machine. A descriptor is none of those things. It is a private note inside one kernel, in one program. The number means nothing anywhere else. A protocol that carried descriptors would break the moment it crossed a network — and Plan 9's whole idea is that a faraway file behaves just like a near one.

So the tool that does everything else in Sophia cannot do the one thing modern Linux graphics runs on.

## Immutable Shared Buffers

Sophia's production CPU image architecture is **Immutable Shared Buffers (ISB)**. The app finishes the pixels, makes their backing immutable, and passes the descriptor over a small local side socket beside 9P. Sophia validates the pixels and retains that backing while it is needed. 9P carries the admission, control messages and lifetime events.

The current Linux implementation uses a *sealed memfd*: a memory-backed file sealed against writing, growing and shrinking, with the seal set itself sealed. “Sealed” describes this mechanism; ISB names the architecture. An unchanged accepted image can remain available to later candidates. Changed content gets a new immutable resource.

We compared ISB with sending pixels as byte records through 9P. The byte implementation remains during migration and in our benchmark comparator. It is being retired from production: clients must support ISB, and there will be no automatic byte fallback or image-size-based transport selection. If the required channel is missing, image setup fails explicitly. If it closes, new image admission stops for that connection epoch; recovery needs a fresh authenticated epoch.

ISB is strict about order. The app asks first. Sophia reserves an offer slot and replies with ImageAdmitted. Only then may the app send the descriptor. Accepted followed by SourceReleased settles the offer; Sophia still holds the immutable backing. ResourceReleased is a separate event after retirement and the last consumer's release. Channel loss never authorizes destroying backing that is still in use.

## The ISB path

```text
              SOPHIA ISB IMAGE PATH     (negotiated shell r9 / lock r2)

 +------------------------------------------------------------------------+
 | SHELL COMPONENT / LOCK PROVIDER  (supervised, sandboxed peer)          |
 |                                                                        |
 | Prepare pixels -> fresh memfd -> seal WRITE | GROW | SHRINK | SEAL     |
 | Send its descriptor only after ImageAdmitted.                          |
 +-------------+------------------------------------+---------------------+
               |                                    |
       9P2000.L control/events              SEQPACKET side socket
               |                                    |
 +-------------v------------------+   +-------------v---------------------+
 | Authenticated role root        |   | Discovered endpoint (image.sock)  |
 |                                |   |                                   |
 | image-channel: endpoint, kinds |   | Hello: 40 bytes, no FDs           |
 | image-limits: budgets          |   |   role, epoch, channel generation |
 | image-formats: formats         |   |                                   |
 |                                |   | Source: 200 bytes + one FD        |
 | Peer -> ImageBegin             |   |   via SCM_RIGHTS (sealed source)  |
 | Peer <- ChannelReady           |   |   header + metadata + plane rows  |
 | Peer <- ImageAdmitted          |   |                                   |
 | Peer <- ResourceStatus         |   | No pixel payload in the packet.   |
 | Peer <- SourceReleased         |   +-------------+---------------------+
 | Peer <- ResourceReleased       |                 |
 | Peer <- ChannelClosed          |                 |
 +-------------+------------------+                 |
               |                                    |
 +-------------v------------------------------------v---------------------+
 | SOPHIA SESSION / ROLE RESOURCE OWNER                                   |
 |                                                                        |
 | Hello: verify peer custody and authenticated epoch -> ChannelReady     |
 | ImageBegin: reserve role budget + journal slots -> ImageAdmitted       |
 | Source: match admission; check seals and exact backing size            |
 |         -> read-only mapping                                           |
 |         -> worker validates premultiplied BGRA under the deadline      |
 |         -> retain mapping as immutable resource backing                |
 |                                                                        |
 | ResourceStatus(Accepted) -> SourceReleased(retained backing)           |
 |   Offer settled; Sophia still owns the backing.                        |
 |                                                                        |
 | Candidates reference the resource -> composition / presentation        |
 | Retirement or revocation stops new references.                         |
 | Last consumer ends -> ResourceReleased (while the journal is live)     |
 +------------------------------------------------------------------------+
```

Read it this way: the left side is talk, the right side is the one heavy thing, handed over once. 9P keeps doing what it is great at — names, rules, order and proof. The side socket does the one thing 9P cannot.

## Copies we can avoid

The current benchmark peer builds pixels in a reusable canvas, then uses the Rust SDK to write them into a fresh memfd with `pwrite`. That still transfers the pixels from the canvas into the shared backing. A mapped Rust builder would let the peer draw directly into that backing; this remains an optimization to implement and measure. The C builder already exposes a writable mapping and unmaps it before sealing. In either case, writable shared views must be gone before the buffer is sealed and offered.

Sophia retains the validated mapping instead of making a separate resource-backing copy. Rendering may still upload those pixels to a GPU texture. Neither the ISB name nor passing a descriptor makes that work disappear. We will measure individual stages before changing buffer custody or adding a reusable pool.

## Where this stands

ISB runs through both SDKs and Sophia's real image owners. The latest pilots covered a 1920×24 panel, a 4 MiB upload probe, and a fully presented 1920×512 surface. In the large presented workload, ISB used about 13–14% less combined Session-plus-client CPU and about 53% fewer allocation operations than the IPC comparator. Small-panel CPU and allocation costs remain above IPC.

These were diagnostic pilots on one machine, with two CPU blocks and separate allocation runs. They establish neither statistical repeatability nor a full performance qualification. They support committing to ISB and optimizing its producer, validation and shared control machinery. The existing performance gates remain open.

DMA-BUF remains separate future work for GPU-produced content. Its contract requires a bounded capture into Sophia-owned storage and proven GPU completion before release; it is not a promise of direct scanout or zero-copy composition.
