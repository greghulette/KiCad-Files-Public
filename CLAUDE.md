# KiCad-Files-Public — Working Notes

Every PCB in the droid: body and dome controllers, logics, the WCB, remotes, and
`NaviHiltCore` (the companion board NaviCore firmware runs on). KiCad 10, one directory per
board, a shared custom part library, and no build system — the files *are* the artifact.

**Full engineering documentation is in [`docs/`](docs/README.md). Read the page for the area
you are changing before editing.**

| Working on | Read |
|---|---|
| Anything touching a symbol, footprint, or library path | [docs/LIBRARIES.md](docs/LIBRARIES.md) |
| Editing a `.kicad_sch` / `.kicad_pcb` outside the GUI | [docs/KICAD_EDITING.md](docs/KICAD_EDITING.md) |
| Choosing or specifying a component | [docs/PART_SELECTION.md](docs/PART_SELECTION.md) |
| The NaviHiltCore board specifically | [NaviHiltCore/DESIGN.md](NaviHiltCore/DESIGN.md) |
| The handheld remotes | [R2_HH_Controllers/DESIGN.md](R2_HH_Controllers/DESIGN.md) |

---

## Rules that are easy to break

1. **A project-level `fp-lib-table` entry shadows the global one of the same nickname.**
   Adding a project `01-Custom` entry silently redirects every footprint lookup in that board
   to a local folder that usually holds a *subset* of the real library — parts that resolved
   yesterday stop resolving. This has already broken a board's MCU footprint. Full rules in
   [docs/LIBRARIES.md §2](docs/LIBRARIES.md#2-never-shadow-a-global-nickname).
2. **The custom *symbol* library is not in this repo and is not version controlled.** The
   global `sym-lib-table` resolves `01-Custom` to
   `C:/Users/ghulette/Documents/01-Custom.kicad_sym`, outside any git repo, while the matching
   *footprints* live here at `01-Custom.pretty`. Editing a symbol is an untracked change to a
   file every board depends on. See [docs/LIBRARIES.md §1](docs/LIBRARIES.md#1-where-the-real-libraries-are).
3. **Never edit the `lib_symbols` block embedded in a `.kicad_sch`.** KiCad compares it
   against the on-disk library and raises `lib_symbol_mismatch`. To change a part on one
   board, edit the *instance* fields (Value, Footprint, MPN) only. To change it everywhere,
   edit the on-disk library and let KiCad update the schematic.
4. **A symbol swap must keep pin numbers matching footprint pads.** "Update PCB from
   Schematic" fails outright on a pin/pad mismatch — e.g. a bare-module symbol (pins `1..65`)
   against a devkit footprint (pads `J1_1..J3_22`). Check the pin numbering, not just the pin
   count.
5. **`PWR_FLAG`'s `lib_symbol` must be named `power:PWR_FLAG`.** If the name doesn't match the
   instance's `lib_id`, its pin type parses as `???` and ERC's power-driven check goes wrong
   in a way that looks like a wiring bug.
6. **Verify a part exists and check its real pinout on the vendor page before specifying it.**
   A regulator that was recommended here (`D24V50F3`) does not exist as a product — the 3.3V
   line tops out lower — and the pinout assumed for it was wrong. It was caught at the point
   of ordering. See [docs/PART_SELECTION.md](docs/PART_SELECTION.md).

## Verifying

There is no build. The checks are:

- **ERC in Eeschema** — the bar is **0 errors, 0 warnings**. A warning that is genuinely
  benign gets a note in the board's `DESIGN.md` saying why, not a shrug.
- **DRC in Pcbnew** before any gerber export.
- **Update PCB from Schematic** — the real test that symbols, footprints, and libraries all
  agree. It surfaces mismatches nothing else catches.

Claude cannot run any of these; they are KiCad-GUI actions. After a hand-edit, say what still
needs to be checked in the GUI rather than implying the change is verified.

## Conventions

- **One directory per board**, named for the board. `*-backups/` and `.history/` are KiCad's
  and the editor's, not curated — ignore them when searching.
- **Back up before a hand-edit.** The `.bak`/`.bak6`…`.bak9` files next to a schematic are
  deliberate pre-edit snapshots. Make one and say where it is.
- **The schematic is the source of truth; `DESIGN.md` is the reasoning.** Each board's
  `DESIGN.md` records why the pinout is what it is, what was rejected, and what's still open —
  read it for that. But **confirm any specific pin, net, or part against the `.kicad_sch`
  before relying on it.** The files are the design; the doc is a map of it.
- **A design change updates `DESIGN.md` in the same commit as the schematic**, with a dated
  row in its Revision log. Not every edit qualifies — nudging a component, tidying a label, or
  re-routing a trace changes nothing anyone needs to know. These do: **a pin or net
  reassignment, a part change, a rail or power-budget change, a connector added or removed, an
  RF or layout constraint, and any ERC warning you decide to accept** (say why, so the next
  session doesn't re-investigate it). Also: **anything you got wrong and corrected** — the
  wrong part, the pin that couldn't be used — because the constraint is what stops it
  recurring. Full test in `~/.claude/docs/CONVENTIONS.md` § What earns a doc update.
- **There is no D: drive on this machine.** Any library path starting `D:/` is dead; see
  [docs/LIBRARIES.md §4](docs/LIBRARIES.md#4-known-broken-entries).
