+++
title = "Sophia Display Server"
template = "index.html"
+++

# Sophia

**Sophia** is a modern, transaction-driven X11 display server and compositor. It keeps X11’s flexible application model and adds explicit authority boundaries and synchronized visual commits.

The design is a constitutional cathedral with bazaar edges. The core is a single, coherent system in which no component has unchecked power. The Engine controls pixels and physical input, the protocol frontend applies X11 rules, and the window manager sets layout policy. Well-defined interfaces limit each role.

You can replace or develop everything around this core—window managers, shells, and portal policies—independently. The core enforces safety and presentation; the edges decide how the desktop behaves.

---
