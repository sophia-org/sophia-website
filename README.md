# Sophia Website (`sophia.gg`)

This repository contains the source code, layouts, and technical writings for the official website of the Sophia display server:

👉 **[sophia.gg](https://sophia.gg)**

The site is designed to be extremely lightweight, objective, and fast, honoring the technical aesthetics of the systems hacking community.

---

## Technical Design

I build the site using `zola` (v0.18.0 or greater), a single-binary static site generator written in Rust, allowing the entire site to compile in under 30 milliseconds. To maintain a lightweight initial page weight of under 10KB, I use a single, handcrafted, zero-dependency stylesheet (`static/style.css`) with system-native fonts and an automatic dark/light theme toggle. The aesthetic is inspired by the minimalist, text-centric layout of systems blogs such as `isaacfreund.com`, prioritizing high contrast and readable typography.

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

To test and preview the website on your laptop, install `zola` (v0.18.0 or greater):

```bash
# Arch Linux
sudo pacman -S zola

# macOS / Linuxbrew
brew install zola
```

Run the `zola` watch server:

```bash
zola serve
```

Open **`http://127.0.0.1:1111`** in your browser. Any edits you make to the markdown, stylesheets, or layouts will auto-reload in real-time.

---

## Deployment

To deploy new changes to production, follow this chronological workflow:
1. Stage your local technical content changes using `git add <file>`.
2. Commit your edits: `git commit -m "doc: update technical specs"`
3. Push the commits to the main branch: `git push origin main`
4. Run the remote deployment command to trigger the compiler: `ssh archvps "bash /var/www/deploy-sophia.sh"`

The deployment script pulls your latest commits, compiles the site using the VPS's native `zola` compiler, and serves the updated static output instantly.
