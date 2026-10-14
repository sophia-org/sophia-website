# Sophia Website (`sophia.gg`)

The source for the Sophia website:

👉 **[sophia.gg](https://sophia.gg)**

It's a small static site: plain text, one stylesheet, no JavaScript framework.

---

## Technical Design

I build the site with `zola` (0.18.0 or later), a single-binary static site generator written in Rust. The whole site builds in about 30 milliseconds. There's one hand-written stylesheet, `static/style.css`, with system fonts and automatic light and dark themes, which keeps the first page load under 10 KB. The look borrows from plain-text systems blogs like `isaacfreund.com`: high contrast and easy reading.

---

## Directory Layout

```text
sophia-website/
├── config.toml         # Site settings and menu
├── static/
│   └── style.css       # The one stylesheet
├── templates/          # Page templates
│   ├── base.html       # Shared layout skeleton
│   ├── index.html      # Home page
│   ├── page.html       # Single post or doc
│   └── section.html    # Blog and docs listings
└── content/            # Markdown articles & guides
    ├── _index.md       # Homepage copy
    ├── docs/           # Documentation
    └── blog/           # Blog posts
```

---

## Local Development

To preview the site, install `zola` 0.18.0 or later:

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

Open **`http://127.0.0.1:1111`**. The page reloads whenever you change the Markdown, the stylesheet or a template.

---

## Deployment

Commit your changes, push them to `master`, then run the deploy script on the server:

```bash
git push origin master
ssh archvps "bash /var/www/deploy-sophia.sh"
```

The script pulls the latest commits, builds the site with the server's `zola` and serves the new output.
