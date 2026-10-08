# KiCad-Files-Public — Engineering Documentation

Written for someone (or some session) starting cold: read the relevant page, then edit — no
prior context required.

---

## Start here

| Document | Read it when |
|---|---|
| **[LIBRARIES.md](LIBRARIES.md)** | Always first, if a symbol, footprint, or library path is involved. The library layout here is genuinely tricky and has broken boards |
| **[KICAD_EDITING.md](KICAD_EDITING.md)** | Editing a `.kicad_sch` or `.kicad_pcb` as text rather than in the GUI — the s-expression rules, what KiCad validates, what to back up |
| **[PART_SELECTION.md](PART_SELECTION.md)** | Choosing a component, building a symbol or footprint for it, or writing a BOM line |

Per-board design notes live with the board, not here — e.g.
[NaviHiltCore/DESIGN.md](../NaviHiltCore/DESIGN.md).

---

## The short version

KiCad 10. One directory per board, ~25 of them. No build system: the `.kicad_sch` /
`.kicad_pcb` files are the deliverable, and gerbers are exported by hand.

Three rules cover most mistakes:

1. **Don't shadow a global library nickname with a project-level one.** It silently redirects
   every lookup on that board to a folder holding a subset of the parts.
2. **Don't edit the `lib_symbols` block inside a schematic.** Edit instance fields, or edit
   the on-disk library — never the embedded copy.
3. **Verify a part exists and check its pinout on the vendor page before specifying it.**

## Verification

There is nothing to run. ERC (target: 0 errors, 0 warnings), DRC, and "Update PCB from
Schematic" are GUI actions, and "Update PCB" is the one that actually proves symbols,
footprints, and libraries agree.

A hand-edited schematic is **not verified** until it has been opened in KiCad. Say that
rather than implying otherwise.

## Keeping documentation honest

**The KiCad files are the source of truth.** These pages tell you where to look and why a
decision was made. They are not evidence of what a net or a pin is *today* — confirm that in
the `.kicad_sch`. A page that disagrees with the files is a bug, and gets fixed in the change
that revealed it.

Page bodies describe how things work **now** — no superseded paragraphs, no "this used to…".
History lives in the **Revision log** at the bottom of each page, added in the same commit as
the change it records. Format: `~/.claude/docs/CONVENTIONS.md` § Revision logs.

---

## Revision log

| Date | Commit | Change | Why |
|---|---|---|---|
| 2026-08-04 | — | Index created alongside the first three pages | Repo had no documentation at all |
