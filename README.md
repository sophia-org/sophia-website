# Sophia Website (`sophia.gg`)

This repository contains the source code, layouts, and technical writings for the official website of the Sophia display server:

👉 **[sophia.gg](https://sophia.gg)**

The site is designed to be extremely lightweight, objective, and fast, honoring the technical aesthetics of the systems hacking community.

---

## Technical Design

*   **Engine:** Built using **Zola**, a single-binary static site generator written in Rust. The entire site compiles in under 30 milliseconds.
*   **Styling:** A single, handcrafted, zero-dependency stylesheet (`static/style.css`) with system-native fonts and an automatic dark/light theme toggle. The entire initial page weight is less than 10KB.
*   **Aesthetic:** Inspired by the minimalist, text-centric design of systems blogs (such as `isaacfreund.com`), prioritizing readable typography and high contrast.

---

## Directory Layout

```text
sophia-website/
├── config.toml         # Site configuration & menu navigation
├── static/
│   └── style.css       # Zero-dependency minimalist theme
├── templates/          # Reusable HTML template frame and layouts
│   ├── base.html       # Shared layout skeleton
│   ├── index.html      # Flowing text homepage
│   ├── page.html       # Individual post & doc viewer
│   └── section.html    # Index of blog posts and documentation
└── content/            # Markdown articles & guides
    ├── _index.md       # Homepage copy
    ├── docs/           # Technical manuals and guides
    └── blog/           # Technical diaries and announcements
```

---

## Local Development

To test and preview the website on your laptop, install Zola:

```bash
# Arch Linux
sudo pacman -S zola

# macOS / Linuxbrew
brew install zola
```

Run the Zola watch server:

```bash
zola serve
```

Open **`http://127.0.0.1:1111`** in your browser. Any edits you make to the markdown, stylesheets, or layouts will auto-reload in real-time.

---

## Deployment

The website is hosted on a custom VPS (`archvps`) running Caddy with automated Let's Encrypt SSL.

To deploy new changes, commit and push your work to GitHub:

```bash
git add .
git commit -m "doc: add new technical post"
git push origin main
```

Then, trigger the automated deployment script on your VPS:

```bash
ssh archvps "bash /var/www/deploy-sophia.sh"
```

The script will automatically pull your latest commits, rebuild the site using the VPS's Zola compiler, and deploy the updated static output instantly.
