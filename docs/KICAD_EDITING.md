# Editing KiCad files as text

KiCad files are s-expressions, and editing them directly is often the fastest way to make a
bulk change — remapping thirty net labels, swapping a symbol, adding `no_connect` flags.
It is also the fastest way to produce a file that opens but is subtly wrong.

This page is what to know before doing it.

---

## 1. Back up first, always

Make a copy before a hand-edit and say where it is:

```
NaviHiltCore.kicad_sch      ← working file
NaviHiltCore.kicad_sch.bak  ← pristine pre-edit snapshot
```

The `.bak`, `.bak6`…`.bak9` files next to a schematic are deliberate snapshots at points
worth returning to, not clutter. KiCad's own `*-backups/` folders and the editor's
`.history/` are separate and automatic — don't rely on either as *your* backup, and skip
them when searching.

## 2. Instance fields vs. `lib_symbols`

Every `.kicad_sch` embeds a `lib_symbols` block: a snapshot of each symbol's definition,
copied from the on-disk library at placement time. KiCad compares it against the library and
raises **`lib_symbol_mismatch`** when they differ.

| Goal | Edit |
|---|---|
| Change a part on **this board only** (Value, Footprint, MPN) | the **instance** — the `(symbol (lib_id …))` block and its properties |
| Change a part **everywhere** | the **on-disk library**, then let KiCad update the schematic |
| — | **never** the embedded `lib_symbols` definition |

Editing the embedded definition is what triggers the mismatch, and it desynchronises the
board from the library in a way that resurfaces at the worst moment.

An unused `lib_symbol` left in the block after a symbol swap is harmless — don't clean it up
just because it looks stale.

## 3. Swapping a symbol

The failure mode: "Update PCB from Schematic" refuses to run because pin numbers don't match
pad numbers.

A bare-module symbol numbers pins `1..65`. A devkit footprint numbers pads `J1_1`…`J3_22`.
Both are "an ESP32-S3", both have plausible pin *counts*, and they are completely
incompatible. **Check the numbering scheme, not the count.**

When swapping:

1. Pick a symbol whose pin numbers are the footprint's pad names.
2. Re-map every net label and `no_connect` to the new pin positions. GPIO→net assignments
   stay the same; the *pins they land on* change.
3. Don't forget power pins — a devkit fed from 5V needs its 5V pin wired, which a bare module
   doesn't have.
4. Run "Update PCB from Schematic" in the GUI. That is the only real proof.

## 4. ERC

The bar is **0 errors and 0 warnings**. Warnings that are genuinely benign get a note in the
board's `DESIGN.md` explaining why — not a shrug, because next time nobody will remember
which warnings were expected.

Things that reliably confuse ERC here:

- **`PWR_FLAG` naming.** The `lib_symbol` must be named `power:PWR_FLAG` to match the
  instance `lib_id`. If it doesn't, the pin type parses as `???` and the power-driven check
  misfires — it looks like a wiring bug and isn't.
- **Two power sources on one net.** A regulator output typed `power_out` and a `PWR_FLAG` on
  the same rail conflict. Once a real `power_out` drives a rail, remove that rail's
  `PWR_FLAG`.
- **Wrong pin types on a custom symbol.** A symbol whose pins are all `bidirectional`
  produces a spray of `pin_to_pin` warnings wherever it meets anything. Fix the symbol's pin
  types — but if the symbol is shared with other boards, changing them ripples, so check
  who else uses it first.
- **Intentionally-open pins need `no_connect` flags.** Unused straps, PSRAM pins, spare
  GPIOs. A `no_connect` on a spare is also a marker: delete the flag to reuse the pin.

## 5. What Claude can and cannot verify

Claude can edit the files and reason about the netlist. Claude **cannot** run ERC, DRC, or
"Update PCB from Schematic" — all three are KiCad GUI actions.

So after a hand-edit, report what was changed and state plainly that it still needs to be
opened in KiCad and checked. Don't describe an unopened edit as verified.

---

## Revision log

Newest first. One row per change to the editing rules.

| Date | Commit | Change | Why |
|---|---|---|---|
| 2026-08-04 | — | Page written — backup discipline, instance-vs-`lib_symbols`, symbol swaps, the ERC traps | Learned the hard way on NaviHiltCore; previously undocumented |
