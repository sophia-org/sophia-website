+++
title = "Engine speaks no client protocol"
date = 2026-09-19
[extra]
author = "niltempus"
+++

Tie a compositor to one application protocol and the two grow together. An X server collects drivers, client state, input routing and composition in one unsupervised address space. A Wayland compositor collects layout, panels, presentation timing and display driving in one loop. In both cases a new protocol means surgery on the program that keeps the screen alive, and I did not want to build that program twice.

So Sophia's Engine starts from a narrower premise. Engine speaks no client protocol at all.

## What Engine knows

Engine knows buffers, damage, geometry and timing. It owns the input devices and the display, keeps the scene graph, commits a window's new size and its new pixels together, renders the result and schedules the page-flip through DRM/KMS. It has never heard of an X window hierarchy, an atom, a selection or a grab. Those belong to somebody else, and in Sophia they do.

That ignorance is the point. A resize, a new buffer and a request for the next frame mean the same thing to Engine no matter which program sent them or how the request arrived. The protocol lives outside, where it can be checked, refused and restarted without touching the frame on screen.

## Who speaks for the clients

The X authority speaks X11. It is a new server, written from scratch in Rust, and it is a full frontend rather than a compatibility layer. It accepts the classic socket, keeps each client's resources inside that client's namespace, checks the request and passes what survives to Engine as a transaction. X clients keep their DRI3 descriptors and shared memory. The server implements part of X11 today, and compatibility work is still going on. None of that reaches Engine in X's own terms.

The desktop roles speak 9P, and they are not frontends. The window manager, the shells, output, the screen lock, factotum, read-only inspection and the portals each hold one export on a single 9P2000.L core, with its own endpoint, file tree and permissions. They share the server code, the identity model and one firm rule: no file descriptors in the 9P stream, ever. Descriptors cross on a separate channel.

A native application frontend will speak 9P as well, once the shared contract between the frontends and Engine is written. That contract is the real boundary. Today's X wiring is its first user, not the contract itself, and until the contract exists there is no second frontend to plug in. I am not going to pretend otherwise in the diagram.

## What this does not promise

It does not promise a Wayland frontend. There is none, and I am not planning one. When the application frontend lands, someone else can write a Wayland layer as a 9P client of it and carry each Wayland client as a sub-identity with its own tree. That is a door left open, not work on my list.

Nor does it promise that a new protocol costs a weekend. A frontend still has to keep its clients' resources straight, check every request against the right namespace and turn the results into honest transactions. Engine's neutrality removes the need to rebuild the compositor. It does not remove the need to write the frontend.

What it does promise is smaller and, to me, worth having. The program that owns your screen does not parse your applications' wire protocols. The program that does can die without killing Engine. Replacing it, or writing the next one in another language, waits on the shared contract — which is why that contract matters.
