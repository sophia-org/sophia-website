+++
title = "Moving Pixels, Take Two"
date = 2026-10-29
[extra]
author = "niltempus"
+++

On October 18, this blog said sealed files would move Sophia's pixels. The design then changed. This post says what changed, what did not, and why.

The rule did not change: once Sophia accepts an image, no client can change it. The way we keep that rule did.

## What sealed files got right, and what they cost

The first design worked like this. For each new image, the shell made a fresh file in memory. It filled the file with pixels. It sealed the file, so the pixels could not change. It sent Sophia the file handle. Sophia kept that exact file.

That proof was strong. It was also new work on every frame: make a file, size it, map it, seal it, send it, check it. At 120 frames a second, small costs cease to be small. Our tests showed the large-image path could win, but the small panel path still cost too much CPU.

So we kept the rule and changed where the safe copy lives.

## Two trays, used again and again

The CPU path now uses a small ring. Think of two fixed trays. The client and Sophia agree on the number of trays and the size of each tray at the start. Nothing about that deal can grow later.

For each frame:

1. The client fills one free tray with pixels.
2. It writes a short note to Sophia: here is a full tray; these are its size and name.
3. Sophia copies just those bytes into storage it owns and checks them.
4. If the copy works, Sophia says the image is accepted, and then says the tray is free.
5. The client may reuse that tray only after the second message.

If any step fails, the client treats the tray as held. It never guesses that a tray is free. A lost answer means "do not touch it," not "try again on the same tray."

Here is the trick in one line: the tray itself is not locked. It does not need to be. Sophia does not display the tray. It displays its own copy, and its copy cannot be reached by the client. Sealing a new file on each frame bought safety we can now get from one copy and a clear order of events.

## Drawing straight into the tray

One copy on the client side can still go away. Today a shell may draw its pixels in one place and then copy them into a tray. The better shape is to draw right into the tray. The SDK can now do that, and a small test client does it. We are now moving our shells to that shape. Until that work passes its tests, we claim no speed gain for it.

## The GPU lane

Some shells have leave to use the GPU. They should not have to copy their work back to the CPU just to show it.

So there is a second lane. The shell draws on the GPU. It hands Sophia a finished GPU image through a Linux graphics handle. Sophia checks that the image came from the same GPU and uses a form it can read. It sets aside its own GPU space first. Then it copies the image into that space on the GPU and waits for proof the copy is done.

Only then may the shell reuse its image.

Note the words: Sophia still makes a copy. We say so in plain sight. This path does not promise that the shell's image goes straight to the screen, and it does not promise "zero copy." It promises something we can prove: Sophia owns the copy it shows.

This GPU lane is still classed as test work. It is turned on only in set test builds.

## Both lanes end on the same shelf

```text
  CPU SHELL                          GPU SHELL
  draw into one free ring tray       draw into a GPU image
             |                                  |
  short offer note over 9P           image handle + offer over 9P
             |                                  |
             v                                  v
  +--------------------------------------------------+
  | SOPHIA                                           |
  | copy the image into space Sophia owns,          |
  | wait for proof the copy is done,                |
  | then tell the shell its tray or image is free    |
  +-----------------------+--------------------------+
                          |
              one accepted image that
              no client can now change
                          |
      shell asks leave, places the image,
      Sophia draws the frame and flips it
                          |
              image freed only after
              its last use has ended
```

9P still carries the talk: names, rules, order, and events. Handles still never travel on 9P. They use the small side path made for them.

## Where this stands

The ring is now the one CPU pixel path for native shells. There is no fall-back switch, and the old sealed shell path is refused. (The screen lock kept its sealed path; it has its own rules.) The GPU lane is real but still under test. Drawing right into the tray is the next piece of work, in front of a later GPU step: drawing right into the export image, so the shell skips one GPU copy too.

We will measure each step on its own. A faster fill will not be called a faster shell, and a GPU copy saved will not be called a faster desktop, until the whole path proves it.

The October 18 post stays as written. It was true when we wrote it. This one is true now.
