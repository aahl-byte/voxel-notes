# voxel-notes — house style & conventions

These are living study notes on **voxels and voxel algorithms**, published as a
no-build docsify site rooted at `notes/`. This file is the source of truth for how
the site is organized and written; keep edits consistent with it.

The product is **UNDERSTANDING, not coverage.** A page can be perfectly accurate and
still fail if a reader finishes it with no mental model. Accuracy is necessary, not
sufficient.

## teach RIGHT → LEFT

Open every page with the OUTCOME — what a real system or person is trying to do —
then work leftward into the capability, then the specifics. Never lead with a
primitive or with code. Describe the mechanism in plain language FIRST, then attach
the standard term to what you just described. Don't substitute an unrelated metaphor
for the real description.

## structure like an onion (two scales)

- **Macro — domains** (sidebar top level), in dependency order: VOXEL FOUNDATIONS is
  the global foundation; each later domain rests on the earlier ones. The spine
  follows the life of voxel data: foundations → storing → generating → surface
  extraction → rendering → engines → optimization → simulation → applications.
- **Micro — phases** within a domain: foundation (mental model) → building blocks
  (the moving parts) → cross-cutting (tradeoffs, "use X instead of Y because Z") →
  synthesis (worked examples, real systems). Emit a phase only when it holds pages.

A reader should be able to stop after any domain or any page and still hold a true,
if coarse, model.

## house style

- Markdown, one topic per file under a domain folder.
- **First line of every content page** is exactly:
  `<link rel="stylesheet" href="./css/globals.css">`
- `<em>...</em>` is a COLORED HIGHLIGHT for the key phrase in a definition — not
  italics, not ordinary emphasis.
- Lowercase, casual headers: `#` page, `##` section, `###` sub-topic, `####` finer.
- Bullets over prose: a one-line plain-language summary, then bullets.
- Code is short and illustrative, and comes AFTER the concept — never lead with it.
- **Contrast is where understanding lives:** prefer "X instead of Y because Z" and a
  "when to use" list wherever options compete (meshing algorithms, storage
  structures, render paths — the choice is the lesson).

## linking

- Note-to-note links are RELATIVE: `./other.md`, `../section/page.md`.
- Nav files (`_sidebar.md`, `_navbar.md`, `_coverpage.md`) use ABSOLUTE paths: `/section/page.md`.
- Cross-link liberally: point each page back to its prerequisite chapter(s), and link
  forward where an idea is owned by another page (e.g. SVDAG appears in both storing
  and optimization — link them).

## sources & papers (this site's convention)

- Cite **open-access** papers inline where a claim or algorithm has a canonical
  reference (e.g. marching cubes → Lorensen & Cline; dual contouring → Ju et al.;
  ESVO → Laine & Karras; SVDAG → Kämpe et al.; transvoxel → Lengyel; VDB → Museth;
  voxel cone tracing → Crassin et al.).
- Keep a local PDF of every referenced open-access paper under `papers/` so primary
  sources travel with the notes. Link both the local PDF and the original URL.
- Use the `/deep-research` skill when authoring or expanding a page to ground it in
  multiple sources before writing.

## verification (before publishing)

Run `node <notes-architect>/scripts/verify.js notes`. It enforces: every page's first
line is the stylesheet link, every relative cross-link resolves, and every page is in
`_sidebar.md` with no orphans. Fix all findings before committing.

## current status

Skeleton stage: all 52 content pages are **placeholders** stating their intended
scope. The architecture (domains, phases, sidebar) is in place. Next step is
authoring page bodies — one focused page at a time, in house style, with sources.
