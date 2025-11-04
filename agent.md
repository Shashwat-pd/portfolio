# Shashwat Portfolio – Agent Notes

These notes capture the current state of the repo, decisions we made, how things work, and what to do next.

## Overview
- Minimal one‑pager with muted black/gray styling and compact typography.
- Intended font: self‑hosted FiraCode Nerd Font (Regular). Fallbacks in place if font files are missing.
- Sections: Header + Resume, Projects (collapsible), Writeups (legend tooltip), XKCD figure, Elsewhere.

## Files & Structure
- `index.html`: main page.
  - Header: name + tagline; Resume control with animated gradient ring; stronger rule below.
  - Projects: full‑row links; “View more/less” toggle reveals `.more` items.
  - Writeups: full‑row links; ℹ️ tooltip shows legend “⚙️ Technical · ☕ Casual” on hover/focus; inline legend hidden by default.
  - XKCD Figure: Brain Upload after Writeups with attribution.
  - Elsewhere: social and email links (placeholders).
- `styles.css`: global typography, layout, resume gradient ring animation, list toggles, tooltip, content styles for writeups.
- `assets/fonts/`: place font files here.
  - Expected: `FiraCodeNerdFont-Regular.woff2`, `FiraCodeNerdFont-Regular.woff`.
- `assets/images/`: images and figures.
  - Expected: `brain_upload.png` (xkcd #1666). A `.keep` exists.
- `writeups/hello-world.html`: example writeup page.
- `writeups/template.html`: copy to create new writeups.

## Header & Resume
- Tagline: “Backend Practitioner. Sucker for Vim Motions”.
- Resume button:
  - Click: opens `resume.pdf` in a new tab (desktop) or same tab (mobile for reliability).
  - Alt/Option‑click (desktop): direct download.
  - Keyboard: Alt+Enter or Alt+Space downloads.
  - Mobile: double‑tap to download; single tap opens.
  - Hint: “Click to view · Alt/Option‑click to download” (desktop) or “Tap to view · Double‑tap to download” (mobile). Always visible on small screens.
  - Gradient ring: conic gradient border animates in a slow spin; respects `prefers-reduced-motion`.
- Strong rule: `<hr class="rule-strong" />` below resume for clearer separation.

## Projects
- Mark additional items with `li.more` to hide them when collapsed.
- Toggle button (`.more-toggle`) switches between “View more” and “View less”.
- Each item is a single anchor: title + one‑liner are fully clickable.

## Writeups
- Header has an ℹ️ button and an inline legend span. The legend appears only on hover/focus of the ℹ️.
- Item icons:
  - Technical: ⚙️
  - Casual: ☕
  - Implemented as emojis to avoid Nerd Font dependency; easy to swap to NF glyphs once fonts are present.
- Add writeups by copying `writeups/template.html`, then link them from `index.html`.

## Writeup Pages
- No date/author; just `<h1>` and content.
- Shared styles via `../styles.css`.
- Extras included in template: headings, links, inline code, code blocks, blockquote, figure.

## Fonts
- Self‑host FiraCode Nerd Font (Regular): place files in `assets/fonts/` and keep the same filenames (or update `@font-face`).
- The site uses the Nerd Font for everything (monospace aesthetic). `.mono` class remains for emphasis where needed.

## XKCD Figure
- File: `assets/images/brain_upload.png` (not committed yet).
- Attribution (shown under image):
  - “Brain Upload” by Randall Munroe — xkcd #1666 — https://xkcd.com/1666/ — CC BY‑NC 2.5
- If you prefer, we can hotlink to `https://imgs.xkcd.com/comics/brain_upload.png` instead of storing locally.

## Customization Notes
- Colors: adjust `:root` vars in `styles.css` (`--text`, `--muted`, `--line`).
- Gradient animation:
  - Implemented with CSS `@property --ang` and a spinning `conic-gradient`.
  - Adjust speed in `.btn { animation: spin 8s linear infinite; }`.
  - Disable or set to hover‑only if preferred.
- Section spacing: `main > section + section { margin-top: 28px; }`. XKCD has larger margins.
- Tooltip behavior: writeups legend shows on hover/focus of ℹ️ only.

## TODOs / Next Steps
- Add `resume.pdf` to the repo root.
- Add font files to `assets/fonts/` (see Filenames above).
- Add `assets/images/brain_upload.png`.
- Replace all placeholder links in Projects, Writeups, and Elsewhere.
- Optionally switch writeups icons to Nerd Font glyphs once fonts are present.
- (Later) Hosting on GitHub Pages as planned.

## Known Limitations
- iOS may open PDF inline rather than “downloading” it; saving is via the share sheet.
- The double‑tap download window is ~220ms; can be tuned.
- `@property` and animated `conic-gradient` require modern browsers; reduced‑motion is respected.
- Without NF font files, emojis are used for icons and the font falls back to system monospace.

## How to Add a New Writeup
1. Copy `writeups/template.html` to `writeups/my-title.html`.
2. Update `<title>` and the `<h1>`.
3. Add your content in the `.content` div.
4. Link it from `index.html` under Writeups (add as a new list item; mark as `tech` or `casual`).

If you want automation (a small script to scaffold writeups), say the word.
