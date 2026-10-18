+++
title = "Moving Pixels with 9P2000.L"
date = 2026-10-18
[extra]
author = "niltempus"
+++

9P is good at almost everything Sophia asks of it. It carries files, names, permissions, control messages and events. It is how the window manager, the shells, the screen lock and the other parts of the system talk to the session.

Pixels are the one hard case.

## The problem is not speed. It is shape.

On Linux, fast graphics does not move bytes. It moves *descriptors*.

A descriptor is a handle to a piece of memory that the kernel manages. One program renders a frame into that memory and hands the descriptor to another program. The second program looks at the same memory. Nothing is copied. This is how Wayland works, and it is how a buffer can reach the screen hardware without ever being copied at all.

9P cannot hand over a descriptor. It only moves bytes.

There are good reasons for this. 9P was made to work the same over a cable, a pipe or a socket, on any machine. A descriptor is none of those things. It is a private note inside one kernel, in one program. The number means nothing anywhere else. A protocol that carried descriptors would break the moment it crossed a network — and Plan 9's whole idea is that a faraway file behaves just like a near one.

So the tool that does everything else in Sophia cannot do the one thing modern Linux graphics runs on.

## Plan 9 never had this problem

In Plan 9, graphics lives in the kernel, and the kernel owns the screen. When a program uploaded its pixels, the bytes landed right next to the display. After that, programs only traded small commands: draw this picture here. The pixels stayed put.

Sophia has no such luck. On Linux, the kernel owns the screen hardware, but a normal user program — the compositor — decides what goes where. Each app renders alone, in its own memory, behind its own walls. A frame is two locked doors away from the glass. Bytes can be pushed through those doors, but each push means a copy. At screen size, sixty or a hundred and twenty times a second, copies stop being cheap.

## Two ways across

Sophia keeps 9P for control in both cases. Only the pixels change route.

- **Bytes.** The app sends the frame as records in 9P, and the session puts the bytes together on its side. This is simple, and it works everywhere 9P works. It also copies every byte, every frame.
- **Sealed.** The app renders into a special piece of memory (a *memfd*), then *seals* it: frozen, shrink-proof, safe to share. The descriptor for that memory crosses on a small side socket, next to 9P — never in it. The session maps the same memory, checks it, and shows it. The frame is not copied into a second buffer — though making the sealed memory still costs work, so this is not free from end to end.

The sealed path is strict about order. The app asks first. The session saves one of a few offer slots and says yes. Only then may the app send the descriptor. When the last user of the frame lets go, the session says so, once. If the side channel is missing, the session can fall back to bytes when the caller allows it. If the channel closes mid-run, descriptor admission stops for the rest of that run.

## The sealed system

```text
              SOPHIA SEALED IMAGE PATH  (negotiated shell r9 / lock r2)

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

## Where this stands

This is built and under test, not shipped as a speed claim. The sealed path now runs end to end in our own tests, and we are measuring it against the byte path at a large frame size (1920×512, about 3.9 MB a frame) — the size where copies should start to hurt. Small frames mostly measure message overhead, and we do not want to fool ourselves with an easy win.

DMA-BUF — handing the screen hardware the buffer itself, with no mapping step at all — comes later, and only if the numbers say it is worth it.

The lesson is narrow, and it is the good kind: a protocol can be right for a whole system and still need one small door in its side. Build the door. Keep the house.
