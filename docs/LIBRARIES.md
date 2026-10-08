# Libraries

Where symbols and footprints actually come from, and the several ways that goes wrong here.
This page has cost more debugging time than everything else in the repo combined — read it
before touching a symbol, a footprint, or a `*-lib-table`.

---

## 1. Where the real libraries are

Custom parts are split across two different places, and **only half of it is in this repo**:

| | Nickname | Resolves to | In git? |
|---|---|---|---|
| **Symbols** | `01-Custom` | `C:/Users/ghulette/Documents/01-Custom.kicad_sym` | ❌ **no** |
| **Footprints** | `01-Custom` | `<repo>/01-Custom.pretty` (repo root) | ✅ yes |

Both are registered in the **global** tables at
`C:/Users/ghulette/AppData/Roaming/kicad/10.0/{sym,fp}-lib-table` — not per project.

The consequence: **editing a custom symbol is an untracked change to a file every board
depends on.** There is no diff, no history, and no way to see what changed. Back it up
(`.bak` alongside) before editing, and say in your summary that the change landed outside
version control.

> There is a second, stale copy at `<repo>/Custom Libraries/01-Custom.kicad_sym`. It is
> **not** what KiCad loads, and it has diverged from the live file — confusingly it is the
> *larger* of the two, so size is not a guide to which is current. Do not edit it expecting
> an effect, and do not treat it as a backup of the live library.

## 2. Never shadow a global nickname

A project-level `fp-lib-table` entry **overrides** a global entry with the same nickname for
that project only.

This has already broken a board. Adding a project `01-Custom` entry pointing at a
project-local `01-Custom.pretty` redirected every footprint lookup on that board to a folder
holding only *some* of the parts — an MCU footprint that had resolved fine for weeks
suddenly "wasn't found", while the symbol, the schematic, and the global table were all
still correct.

**Rule: don't add a project-level entry for a nickname the global table already defines.** If
a footprint is missing, the fix is almost always to correct the *component's* `Footprint`
field to a valid `nickname:name`, not to add a library.

Related: a `Footprint` field with a bogus nickname (`E22-900M22S:E22-900M22S`) presents
exactly like a missing library. Check the field before concluding a library is absent.

## 3. Changes need a library reload

KiCad reads the library tables at startup. After editing any `*-lib-table`, a footprint will
keep reporting "not found" until you reload libraries or restart KiCad. If an edit looks
correct and still fails, restart before investigating further.

## 4. Known broken entries

These are live problems, not history:

- **`NaviHiltCore/fp-lib-table` points at `D:/GitHub/KiCad-Files-Public.pretty`.** There is
  no D: drive on this machine, so the entry is dead. It appears to be unused (the board's
  footprints resolve through the global `01-Custom`), but it will confuse anyone reading it
  and would break on a machine that *does* have a D:. Safe to delete.
- **`R2_HH_Controllers/fp-lib-table` defines `01-Custom` → `${KIPRJMOD}/01-Custom.pretty`** —
  a shadow of the global nickname, exactly the pattern in §2. It works only because that
  board's parts happen to be in the local folder.
- **The global `fp-lib-table` contains a `${KIPRJMOD}`-relative entry**, `Library` →
  `${KIPRJMOD}/01-Custom.pretty/Library.pretty`. A project-relative path in a *global* table
  resolves differently in every project; that folder exists only under `NaviHiltCore`, so the
  entry dangles for every other board.

None of these is urgent. All three are worth fixing the next time that board is opened.

## 5. Scattered `.pretty` folders

There are ~16 `*.pretty` directories in the repo and ~40 loose `.kicad_sym` files. Most are
vendor downloads kept where they landed (`Custom Libraries/<part>/...`), single-part
libraries, or per-board copies.

**`<repo>/01-Custom.pretty` at the repo root is the central footprint library** — the one the
global table points at. Everything else is either a per-board convenience copy or an
unimported download.

When adding a part that more than one board will use, put the footprint in the root
`01-Custom.pretty` and the symbol in the live `01-Custom.kicad_sym`. Don't create a new
nickname for a one-off part.

## 6. Checklist for adding a custom part

1. Confirm the part exists and its pinout is right — see [PART_SELECTION.md](PART_SELECTION.md).
2. Symbol → live `01-Custom.kicad_sym` (back it up first; it is not in git).
3. Footprint → repo-root `01-Custom.pretty`.
4. **Give pins correct electrical types** (`power_in`, `power_out`, `input`,
   `open_collector`, …). Getting these right is what makes ERC come back clean instead of
   producing warnings you learn to ignore.
5. Pin *numbers* in the symbol must match pad numbers in the footprint. This is what
   "Update PCB from Schematic" checks, and the only thing that catches a mismatch.
6. Restart KiCad if you touched a library table.
7. Run ERC and "Update PCB from Schematic" in the GUI before calling it done.

---

## Revision log

Newest first. One row per change to how libraries are actually organised or resolved.

| Date | Commit | Change | Why |
|---|---|---|---|
| 2026-08-04 | — | Page written. Recorded the symbol/footprint split, the nickname-shadowing rule, and three broken table entries found while surveying | These traps had cost real debugging time and lived only in session notes |
