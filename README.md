# voxel-notes

Living study notes on **voxels and voxel algorithms** — how a 3D grid of cells is
stored, generated, meshed, rendered, optimized, simulated, and applied. Built as a
no-build [docsify](https://docsify.js.org) site under [`notes/`](./notes), structured
as a two-scale onion (9 domains, each peeling from a coarse mental model inward to
specifics).

## read it

- **Deployed:** GitHub Pages (see `.github/workflows/deploy-pages.yml`).
- **Locally:** `make serve` (or `cd notes && python3 -m http.server 3000`), then open
  the printed URL.

## what's inside

- `notes/` — the self-contained docsify site: 9 domains, 52 pages.
- `notes/papers/` — local PDFs of the open-access papers cited across the notes, so
  the primary sources travel with the repo. Pages link both the local PDF and the
  original source.
- `CLAUDE.md` — house style and conventions for editing the notes.
- `Makefile` — `make serve` / `open` / `stop` / `verify`.

The domains, in dependency order: **foundations → storing → generating → surface
extraction → rendering → engines → optimization → simulation → advanced applications.**
