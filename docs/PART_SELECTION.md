# Choosing and specifying parts

A wrong part number is the most expensive mistake available in this repo. Software errors get
recompiled; a wrong part gets ordered, arrives, doesn't fit, and costs a board spin.

---

## 1. Verify before recommending

**Never specify a part number from pattern-matching a family.** Confirm on the vendor's own
page that:

1. **The part exists.** Not the family — that exact ordering code.
2. **It meets the actual spec** at the actual operating point.
3. **The pinout is what you think**, read off the vendor's drawing.
4. **The mechanical outline matches** the footprint you're about to use or build.

### The case this rule comes from

A 3.3V, ~1.3A-peak rail needed a buck regulator. `D24V50F3` was specified by reasoning that
Pololu's D24V50 family (5A) would have a 3.3V variant, the way most families do.

It doesn't. The D24V50 line is 5V and up; Pololu's high-current 3.3V options step from
D24V25F3 (2.5A) to D24V150F3 (15A). The correct part was **D24V22F3** (#2857, 3.3V 2.6A).

Worse, the footprint borrowed from the 5V board had a different pin order —
`En·VIN·GND·GND·VOUT` against the real part's `PG·EN·VIN·GND·VOUT`. A board built to that
would have put enable where power good goes.

It was caught at the point of ordering, by Greg, not by the design. **Assume the same class
of error is present in any part you specify from memory.**

## 2. Building the symbol and footprint

Once the part is confirmed:

- **Pin electrical types matter.** `power_in`, `power_out`, `input`, `output`,
  `open_collector`, `bidirectional`. Correct types are the difference between ERC coming back
  clean and ERC producing noise that gets ignored — which is how real errors hide.
- **Pin numbers must equal footprint pad numbers.** See
  [KICAD_EDITING.md §3](KICAD_EDITING.md#3-swapping-a-symbol).
- **Mounting holes are part of the footprint.** If you omit them, say so explicitly — they're
  easy to not notice missing until the board is fabricated.
- Put shared parts in the central libraries; see
  [LIBRARIES.md §6](LIBRARIES.md#6-checklist-for-adding-a-custom-part).

## 3. Power budgeting

Size rails from **peak** draw, not typical. Two 915MHz LoRa modules that idle at ~20mA draw
~500–650mA each while transmitting; a rail sized for the average browns out mid-packet, which
presents as a mystery link failure rather than a power problem.

Devkit onboard regulators are small — an ESP32-S3-DevKitC's AP2112 supplies ~600mA total.
Anything with a real RF or motor load needs its own regulator, not the devkit's 3V3 pin.

## 4. BOM

Board BOMs are `.xlsx` alongside the board (`NaviCoreRC_BOM.xlsx`), with `KiBoM-master/` in
the repo root as tooling. A BOM line without a verified manufacturer part number is not
finished — that's the field the whole of §1 exists to protect.

---

## Revision log

Newest first. One row per change to the part-selection rules.

| Date | Commit | Change | Why |
|---|---|---|---|
| 2026-08-04 | — | Page written, built around the verify-before-recommending rule | The `D24V50F3` part didn't exist and was nearly ordered; the rule needed to be written down, not remembered |
