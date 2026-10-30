+++
title = "Moving Pixels, Take Two"
date = 2026-10-29
[extra]
author = "niltempus"
+++

On October 18, this blog said sealed files would move Sophia's pixels. The design then changed. This post sets out the new design we are putting in place: what changed, what did not, and why.

The rule did not change: once Sophia accepts an image, no client can change it. The way we keep that rule did.

## What sealed files got right, and what they cost

The first design worked like this. For each new image, the shell made a fresh file in memory. It filled the file with pixels. It sealed the file, so the pixels could not change. It sent Sophia the file handle. Sophia kept that exact file.

That proof was strong. It was also new work on every frame: make a file, size it, map it, seal it, send it, check it. At 120 frames a second, small costs cease to be small. Our tests showed the large-image path could win, but the small panel path still cost too much CPU.

So we kept the rule and changed where the safe copy lives.

## Paper on two trays

At the start, a CPU client makes one block of shared memory. It splits that block into fixed slots. Each slot is what we call a **tray**. The pixel bytes written in a slot are **the paper on that tray**. Our test panel uses two trays.

For each frame:

1. The client draws pixels on the paper in one free tray.
2. It writes a short note to Sophia: here is a full tray; this is its size and name.
3. Sophia makes a photocopy of the paper, checks it, and keeps the photocopy.
4. Sophia then tells the client the tray is free.
5. The client may draw on that paper again only after this message.

If any step fails, the client leaves the tray alone. A lost answer means "do not touch it." It never means "reuse it now."

Sophia shows its photocopy, not the paper. So the client can later change the paper without changing the image Sophia accepted. That is why the tray does not need a lock.

## Drawing right on the paper

CPU shells draw right on the paper. The renderer writes the frame in a free tray, so there is no first drawing in one place and then a copy onto the paper. That removes one copy before Sophia ever sees the frame.

Copying pixels onto the paper stays for shells that already have finished pixels, such as a decoded picture. For a shell that draws its own frame, though, the tray is where the drawing happens.

## The GPU lane

Some shells have leave to use the GPU. They should not have to copy their work back to the CPU just to show it.

So there is a second lane. The shell draws right into the GPU image it will hand to Sophia. It passes that image through a Linux graphics handle. Sophia checks that the image came from the same GPU and uses a form it can read. It sets aside its own GPU space first. Then it copies the image into that space on the GPU and waits for proof the copy is done.

Only then may the shell reuse its image.

Note the words: Sophia still makes a copy. We say so in plain sight. This path does not promise that the shell's image goes straight to the screen, and it does not promise "zero copy." It promises something we can prove: Sophia owns the copy it shows.

## Both lanes end on the same shelf

```text
  CPU SHELL                          GPU SHELL
  draw on paper in a free tray       draw into a GPU image
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

The ring is now the one CPU pixel path for native shells. There is no fall-back switch, and the old sealed shell path is refused. (The screen lock kept its sealed path; it has its own rules.) Drawing right on the paper is the normal CPU shape. The GPU work follows the same idea: draw right into the export image, so the shell skips one GPU copy too.

We will measure each step on its own. A faster fill will not be called a faster shell, and a GPU copy saved will not be called a faster desktop, until the whole path proves it.

The October 18 post stays as written. It was true when we wrote it. This one is true now.
