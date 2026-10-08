# R2_HH_Controllers — Design Notes

The pair of **handheld remotes** for the droid — one for each hand. Each is an
ESP32-S3-MINI-1-N8 with an EBYTE E22-900M22S 915 MHz LoRa transceiver, a thumbstick, eight
buttons, a trim pot, and three NeoPixels, running on a USB-C-charged Li-ion cell.

They talk to [NaviHiltCore](../NaviHiltCore/DESIGN.md), which carries **two** E22s — one
dedicated to each remote.

Both remotes are **panelized on a single PCB** and snapped apart at mousebites.

Read [../docs/LIBRARIES.md](../docs/LIBRARIES.md) and
[../docs/KICAD_EDITING.md](../docs/KICAD_EDITING.md) before editing the schematic.

---

## 1. One schematic, instantiated twice — read this first

`R2_HH_Controllers.kicad_sch` is the root sheet. It contains almost nothing of its own: ten
`01-Custom:mousebites` symbols and **two hierarchical instances of the same file**,
`Navicore_Remote.kicad_sch`.

| Sheet name | Sheet UUID | Reference block |
|---|---|---|
| `Remote_Right_Hand` | `b71836e0-30a5-4838-99d0-23ddb2c10802` | C1–C18, D1–D3, SW1–SW11, U1/U3/U4/U9, IC1, J1/J2, E1, R1–R15, S1 |
| `Remote_Left_Hand`  | `beaf6c75-f0e0-4d11-845d-31b4d420276c` | the same parts, renumbered — C19–C36, D4–D6, SW12–SW22, … |

**Consequences, and they are easy to get wrong:**

- **There is no "left schematic" and no "right schematic."** Editing `Navicore_Remote.kicad_sch`
  changes *both* remotes. A change intended for one hand lands on both.
- **The two hands are electrically identical by construction.** If they ever need to differ,
  that requires splitting the sheet into two files — not editing "the other one."
- **Reference designators live in the `(instances …)` block**, keyed by sheet UUID, not in the
  symbol. The same symbol is `SW8` on the right and `SW19` on the left. Reading a reference
  off the schematic without knowing which instance you're in will mislead you.
- `_autosave-Remote_Left_Hand.kicad_sch` and `_autosave-Remote_Right_Hand.kicad_sch` are
  **stale autosaves from an earlier two-file layout** (June 2026, against a July schematic).
  They are not part of the design. Don't edit them and don't treat them as the left/right
  variants.

## 2. MCU and pin map

**U9 = ESP32-S3-MINI-1-N8** — 8 MB flash, **no PSRAM**. Unlike NaviHiltCore's N16R8, nothing
here consumes GPIO33–37, which is why `BTN_8` can sit on IO33.

Verified against `Navicore_Remote.kicad_sch`. Identical on both hands.

| GPIO | Net | Function |
|---|---|---|
| IO0  | `BOOT`       | boot strap / SW10 |
| IO1  | `JOY_X`      | thumbstick X (analog) |
| IO2  | `JOY_Y`      | thumbstick Y (analog) |
| IO3  | `JOY_SWITCH` | thumbstick press |
| IO4  | `BTN_2`      | |
| IO5  | `BTN_3`      | |
| IO6  | `BTN_4`      | |
| IO7  | `VOL_POT`    | trim pot R5 (analog) |
| IO8  | `NEOPIXEL`   | D1 → D2 → D3 chain |
| IO9  | `VBAT_SENSE` | battery divider tap (ADC1_CH8) — see §6 |
| IO10 | `NSS`        | E22 chip select |
| IO11 | `MOSI`       | E22 SPI |
| IO12 | `SCK`        | E22 SPI |
| IO13 | `MISO`       | E22 SPI |
| IO14 | `RST`        | E22 reset |
| IO15 | `BUSY`       | E22 busy |
| IO16 | `DIO1`       | E22 interrupt |
| IO17 | `BTN_5`      | |
| IO18 | `BTN_6`      | |
| IO19 | `D-_ESP`     | native USB |
| IO20 | `D+_ESP`     | native USB |
| IO21 | `BTN_7`      | |
| IO33 | `BTN_8`      | free here — no octal PSRAM |
| IO34 | `RX_EN`      | E22 receive enable |
| IO35 | `BTN_1`      | moved off IO9 to free an ADC1 channel |
| IO36 | `CHG_STAT`   | BQ24074 `/CHG`, open-drain — see §4 |
| IO37 | `PWR_GOOD`   | BQ24074 `/PGOOD`, open-drain — see §4 |
| IO38 | `TX_EN`      | E22 transmit enable |

**Every ADC-capable pin is now used.** On the ESP32-S3, ADC1 is GPIO1–10 and ADC2 is
GPIO11–20 — all twenty carry a net. Any future analog input needs a digital pin freed from
ADC1 first, exactly as `BTN_1` was moved off IO9.

Spare: **IO26, IO39, IO40, IO41, IO42, IO45, IO46, IO47, IO48, TXD0, RXD0** — all carry
`no_connect` flags. None is ADC-capable. Avoid IO45/IO46 (strapping) and IO26 (SPI flash);
IO39–42 are the external JTAG pins, free to use since the S3 has USB-Serial-JTAG on
IO19/IO20.

## 3. Controls

| Ref | Part | Role |
|---|---|---|
| S1 | `RKJXV122400R` | thumbstick — X, Y, and press |
| SW1–SW8 | `SW_Push` | the eight face buttons |
| SW9 / SW10 | `SW_Push` | Reset / Boot |
| SW11 | `PCM12SMTR` | power switch → `BAT+_SWITCHED` |
| R5 | `3352T-1-103LF` | 10 k trim pot → `VOL_POT` |
| D1–D3 | `WS2812B-2020` | NeoPixel chain on IO8 (`NEOPIXEL` → `D1-D2` → `D2-D3`) |

## 4. Power and charging

```
USB-C (J1, USB4105-GF-A)          CC1/CC2 each 5.1 kΩ to GND (sink)
   └─ U4 USBLC6-2SC6  (ESD on D+/D−)
        └─ U3 BQ24074RGT  (Li-ion charger + power path)
             ├─ BAT ── J2 (S2B-PH JST) ── cell
             └─ OUT ── CHARGER_OUT ── SW11 ── BAT+_SWITCHED
                                                └─ U2 TPS63802 (buck-boost) ── 3.3V_REG
```

Rails: `USB_VCC`, `CHARGER_OUT`, `JST_BAT+`, `BAT+_SWITCHED`, `3.3V_REG`.

### 4.1 Charger configuration

Every value below is computed from the BQ2407x datasheet, not inferred. The constants come
from its Electrical Characteristics table.

| Pin | Part | Formula | Result |
|---|---|---|---|
| ISET | R11 **1.8 kΩ** → GND | `I_CHG = K_ISET / R_ISET`, K = 890 AΩ | **494 mA** fast charge |
| ILIM | R10 **1.62 kΩ** → GND | `I_IN = K_ILIM / R_ILIM`, K = 1550 AΩ | **957 mA** input limit |
| ITERM | R9 **3.0 kΩ** → GND | `I_TERM = K_ITERM × R_ITERM / R_ISET`, K = 0.030 | **50 mA** (10 % of charge) |
| TS | R14 **10 kΩ** → GND | `V_TS = I_NTC × R`, I_NTC = 75 µA | **750 mV** |
| TMR | **no component**, NC flag | — | default 30 min / 5 h |
| CE | GND | — | charger enabled |

**ISET must stay inside 590 Ω – 5.9 kΩ.** 1.8 kΩ is comfortably within it.

**TS is a current-source input, not a divider.** The charger drives `I_NTC` out of the pin
into a resistance to ground and compares the resulting voltage against **V_HOT = 300 mV** and
**V_COLD = 2100 mV**. A fixed 10 kΩ gives 750 mV — mid-band, so charging is always permitted.
This is TI's documented substitute when no thermistor is fitted. **Never feed TS from the
battery** (see the Revision log — a divider from `JST_BAT+` parks it on V_COLD and blocks
charging). Note `V_DIS(TS)` exists only on the bq24072/73, *not* the bq24074, so pulling TS
high does not disable the function on this part.

**TMR takes a resistor or nothing** — 18–72 kΩ programs the timers, tied to VSS disables them,
unconnected gives the defaults. A capacitor is not a valid configuration. Left unconnected
here: 494 mA into a ~1 Ah cell is ~2–2.5 h against a 5 h timeout, and the timers stretch
automatically when DPPM throttles the current, so the default has ample margin while keeping
the protection against a cell that never terminates.

**EN1/EN2 select the input-limit mode**, and this is the trap: the ILIM resistor only applies
in one of four states.

| EN2 | EN1 | Input limit |
|---|---|---|
| 0 | 0 | 100 mA (USB100) |
| 0 | 1 | 500 mA (USB500) |
| **1** | **0** | **set by R10 — this board** |
| 1 | 1 | standby |

**EN2 is driven from `CHARGER_OUT`, not `3.3V_REG`.** `CHARGER_OUT` is live whenever USB *or*
the battery is present, so the ILIM setting holds regardless of the power switch. Driving it
from the switched 3.3 V rail instead would collapse EN2 to 0 with the remote off — giving
USB100 and **100 mA charging exactly when a handheld is normally charged**. `CHARGER_OUT` sits
at ~3.4–4.4 V against a 1.4 V V_IH minimum and a 6 V absolute maximum, so it is a valid logic
high with margin.

`/CHG` (`CHG_STAT` → IO36) and `/PGOOD` (`PWR_GOOD` → IO37) are **open-drain**. They need
pull-ups to `3.3V_REG` or firmware `INPUT_PULLUP`, or they float when inactive. Neither costs
standby current — both only sink while USB is present.

With only 5.1 kΩ CC pull-downs and no CC-voltage sensing, the board cannot tell a 500 mA port
from a 3 A one, so drawing ~957 mA from a legacy host is out of spec. The BQ24074's V_IN-DPM
loop backs off if the input sags, so it degrades gracefully rather than browning out the host.

### 4.2 The 3.3 V rail

**U2 = TPS63802DLAR**, a buck-boost converter. It regulates 3.3 V across the entire Li-ion
range rather than dropping out partway down, and supplies the E22's transmit burst.

VSON-HR (DLA), 10-pin, 2 × 3 mm. Orderable `TPS63802DLAR` / `TPS63802DLAT`.

**Why not an LDO.** A linear regulator cannot produce 3.3 V from an input near 3.3 V, and
`BAT+_SWITCHED` is the raw cell — 4.2 V falling to ~3.0 V. Below roughly 3.6 V an LDO's output
follows the battery down and the radio browns out while the cell still holds useful charge.
Separately, peak draw is E22 transmit ~500–650 mA plus ESP32-S3 ~40–70 mA plus three NeoPixels
up to ~180 mA — 750–900 mA, more than a 600 mA part can supply, and bulk capacitance buffers
microseconds rather than a LoRa burst.

| | LDO | TPS63802 |
|---|---|---|
| Topology | linear | buck-boost |
| Output at 3.3 V | ~600 mA | **2 A** (V_IN ≥ 2.3 V) |
| Input range | needs > ~3.6 V | **1.3–5.5 V** |
| Holds 3.3 V at a 3.2 V cell | ❌ | ✅ |
| Shutdown current | — | 10–600 nA |

At 2 A the NeoPixels can stay on this rail.

```
BAT+_SWITCHED ──┬── VIN(10)          L1(9) ──[ 0.47 µH ]── L2(7)
             [Cin 10 µF]
                GND

VOUT(6) ──┬──────────────── 3.3V_REG
          │
     [R1 511k]
          │
          ├── FB(4)
          │
     [R2 91k]      [Cout 22 µF]
          │             │
         GND           GND
```

| Pin | Name | Fitted as | KiCad pin type |
|---|---|---|---|
| 1 | EN | UVLO tap — §4.3 | `input` |
| 2 | MODE | GND — power-save | `input` |
| 3 | AGND | GND | `power_in` |
| 4 | FB | R37/R39 tap | `input` |
| 5 | PG | R41 100 kΩ → `3.3V_REG` | `output` |
| 6 | VOUT | `3.3V_REG` | **`power_out`** |
| 7 | L2 | L1 (0.47 µH) | `passive` |
| 8 | GND | GND | `power_in` |
| 9 | L1 | L1 (0.47 µH) | `passive` |
| 10 | VIN | `BAT+_SWITCHED` | `input` |

Support parts: **C42** 10 µF (input), **C40** 22 µF (output), **L1** 0.47 µH, **R37** 511 kΩ /
**R39** 91 kΩ (feedback), **R41** 100 kΩ (PG pull-up).

**`VOUT` must be typed `power_out` in the symbol.** SnapEDA's generated symbol types it as a
plain `output`, which ERC does not count as a power source — so `3.3V_REG` reads as driven by
nothing and every `power_in` on the rail (three NeoPixel VDD, the E22 VCC, the ESP32 3V3)
raises *"Input Power pin not driven by any Output Power pins."* Changing the type is the fix;
a `PWR_FLAG` would only mask it. Edit the on-disk `01-Custom.kicad_sym`, never the schematic's
embedded `lib_symbols` block.

**There is no separate thermal pad.** The footprint has exactly ten pads. Heat leaves through
**pad 8 (GND)**, which is 1.3 mm wide against 0.9 mm for its neighbours and reaches further
under the body. Give it a copper pour and stitching vias in layout — it is the only thermal
path off the part.

**R37 = 511 kΩ / R39 = 91 kΩ is TI's own table value for 3.3 V**, against a 500 mV feedback
reference: `91/(511+91) × 3.3 = 0.499 V`. The low-side resistor must not exceed 100 kΩ.

**Capacitor specs are *effective* capacitance, not marked value.** Recommended Operating
Conditions call for C_I ≥ 4 µF (5 µF nom) and, for V_O > 2.3 V, C_O ≥ 7 µF (8.2 µF nom).
MLCCs lose most of their rating under DC bias, so a part marked 22 µF is what delivers ~8 µF
effective at 3.3 V. Buy on case size and voltage rating, not the marking — a 22 µF 0603 6.3 V
lands on the limit where an 0805 10 V does not.

**There is no upper limit on output capacitance** (datasheet, Output Capacitor). `3.3V_REG`
also carries C2 and C8 (22 µF each) left from the LDO era; they stay. Extra bulk only lengthens
soft-start slightly and improves transient response — which is what the E22's burst load wants.

**The inductor needs a saturation rating, not just a value.** Effective inductance must stay
within 0.37–0.57 µH *in circuit*. Peak current limits are 3.5 A (buck) and 4.8 A (buck-boost);
actual peak here is ~1–2 A, so something in the 3 A saturation class leaves real margin. A
0.47 µH rated 1.5 A would sag out of the window during a transmit burst.

**MODE must not float**, and is tied LOW (power-save). PFM is only entered below 550–900 mA
peak inductor current, so it switches to PWM by itself under transmit load. Routing it to a
spare GPIO instead would let firmware force PWM during transmit for a quieter RF supply and
drop to power-save when idle.

### 4.3 Undervoltage lockout

The TPS63802's EN pin has a precise threshold with built-in hysteresis, which gives cell
protection for three passives — **R13** (200 kΩ), **R35** (100 kΩ) and **C33** (220 nF):

```
BAT+_SWITCHED ──[ R13 200 kΩ ]──┬── EN (pin 1)
                                │
                        [ R35 100 kΩ ]   [ C33 220 nF ]
                                │              │
                               GND ───────────GND
```

| | Typ |
|---|---|
| V_THR;EN (rising) | 1.10 V |
| V_THF;EN (falling) | 1.00 V |

With `k = 1/3`: **off at 3.00 V, back on at 3.30 V.** The 1.10 on/off ratio is set by the chip,
not the divider, and that hysteresis is what prevents the collapse-recover-collapse oscillation
a plain comparator would produce.

The chip's *internal* UVLO is 1.25 V falling — far below Li-ion range — which is why cell
protection has to come from the EN divider rather than the part's own lockout.

**The 220 nF is not optional.** A 650 mA transmit burst sags the cell ~100 mV through its
internal resistance; near the trip point that would drop the rail mid-transmission. 220 nF
against the 67 kΩ Thévenin gives a ~15 ms filter that rides through bursts.

Drain is ~12 µA, and only while the power switch is on — zero in storage.

**This is the backstop, not the primary cutoff.** Firmware should act first using
`VBAT_SENSE` (§6): warn near 3.4 V, stop transmitting near 3.2 V. The hardware trip at 3.0 V
exists for when firmware hangs or the remote is left in a drawer. Setting it closer to the
firmware threshold would make it fire during normal low-battery operation instead of acting
as a safety net.

Because the divider sits on `BAT+_SWITCHED`, which is fed from `CHARGER_OUT`, plugging in USB
holds EN high regardless of cell state — so the board runs while charging even from a deeply
discharged cell. It also satisfies the "EN must not be left floating" requirement.

## 5. RF

`IC1` = E22-900M22S, same transceiver NaviHiltCore uses, so the link pairs directly.

`E1` = **`ANT-915-USP410`** — a Linx surface-mount antenna, soldered to the board. This is a
deliberate difference from NaviHiltCore, which brings its two E22s out to **SMA** connectors:
the hilt can carry external whips, a handheld can't.

Layout still needs the usual 915 MHz care — a controlled-impedance feed from the E22 to the
antenna pad, the manufacturer's keep-out honoured, and ground pour kept clear where the
antenna's datasheet requires it. A hand wrapped around the board detunes a chip antenna far
more than it does an SMA whip.

## 6. Battery monitoring

Battery voltage is read on **IO9 (`VBAT_SENSE`, ADC1_CH8)** through a divider off the raw
cell, plus two digital status lines from the charger.

```
JST_BAT+ ──[ R32 470k ]──┬── VBAT_SENSE ──► IO9
                         │
                    [R28 470k]   [C37 1 µF]
                         │            │
                        GND ─────────GND
```

4.2 V → 2.1 V at the tap, inside the ADC's linear region with `ADC_ATTEN_DB_12`.

**Why 470 kΩ and not something lower:** 10 k/10 k would draw 210 µA continuously — about
15 %/month off a 1 Ah cell with the remote switched off. 470 k pairs draw ~4.5 µA.

**The 1 µF is what makes the high-value divider work.** The ESP32 ADC charges an internal
sampling capacitor from the source; a 235 kΩ Thévenin cannot supply that, and the reading is
noise. The cap supplies the sampling charge locally and stays charged, so there is no settling
delay before a read.

**Sense from `JST_BAT+`, not `BAT+_SWITCHED`.** The switched rail sits downstream of the
BQ24074's `OUT`, which is fed from USB whenever a cable is present — measuring there reads the
charger, not the cell.

`CHG_STAT` (IO36) and `PWR_GOOD` (IO37) come from the charger's `/CHG` and `/PGOOD`, giving
charging / charged / USB-present. Both are open-drain and need pull-ups (§4.1).

### Firmware requirements

- **Never sample while `TX_EN` is asserted.** A 500–650 mA transmit burst sags the cell by
  hundreds of millivolts; a sample caught mid-burst reports a good pack as flat.
- **Median of ~9 samples**, not a mean — rejects outliers instead of smearing them.
- **Use the eFuse ADC calibration** (`esp_adc_cal` / `adc_cali_*`). The S3's raw ADC is
  meaningfully non-linear and calibration is stored per chip.
- **Don't map voltage to percentage linearly.** A Li-ion discharge curve is flat through the
  middle, so a linear map reads pinned at 100 % then collapses. Use a piecewise lookup, or
  report voltage plus warn/critical thresholds.
- **Suppress the estimate while charging** — `PWR_GOOD` tells you when. The cell reads high
  under charge and any state-of-charge figure is meaningless until it rests.
- Warn near 3.4 V and stop transmitting near 3.2 V, above the 3.0 V hardware lockout (§4.3).

There is **no battery field in the E22 link payload** to NaviHiltCore yet, so the telemetry
side still needs designing.

## 7. Panelization

The root sheet carries ten `01-Custom:mousebites` (U10–U28, even). Both remotes are fabricated
as one panel and snapped apart.

When editing the PCB, keep the mousebite tabs clear of traces and of the antenna keep-out —
a break-tab through an RF section is very hard to diagnose after depanelization.

## 8. Library note

This project has its own `fp-lib-table` defining **`01-Custom` → `${KIPRJMOD}/01-Custom.pretty`**,
which **shadows the global `01-Custom`** entry. It works only because this board's footprints
happen to be in the local folder. See
[../docs/LIBRARIES.md §2](../docs/LIBRARIES.md#2-never-shadow-a-global-nickname) — the same
pattern broke a footprint on another board. Don't copy it to a new project.

## 9. Open items

- **`R1` is DNP and it is the only thing between the E22's ANT output and the antenna `E1`.**
  Built to this BOM the radio has no RF path. Almost certainly meant to be a 0 Ω link or a
  matching component — resolve before fabrication.
- **`CHG_STAT` and `PWR_GOOD` have no pull-ups.** Both are open-drain and float when inactive.
  Either fit 10 kΩ to `3.3V_REG` or commit to firmware `INPUT_PULLUP`.
- **WS2812B-2020 run from 3.3 V.** Several variants specify a 3.5 V minimum supply; at 3.3 V
  they can be dim or colour-shifted. Check the part's minimum VDD — this resolves itself if
  the buck-boost lands, since the rail then holds 3.3 V properly.
- **Buttons SW1–SW8 have no external pull-ups or debounce**, unlike `JOY_SWITCH` which has
  R2 (10 k) and C5 (100 nF). Fine with firmware `INPUT_PULLUP`, but it is a firmware
  dependency, and the inconsistency is deliberate rather than accidental.
- **Buy the regulator passives on the right ratings**, not just values: 0805 for the 10 µF and
  22 µF so DC bias doesn't drop them under the effective minimum, and Isat ≥ 3 A on the
  0.47 µH inductor (§4.2).
- **Layout: pad 8 is the only thermal path** off the TPS63802 — it needs a copper pour and
  stitching vias (§4.2).
- The project-level `fp-lib-table` shadow (§8) should be removed once the footprints are
  confirmed present in the central library.
- Antenna keep-out and feed impedance need checking against the ANT-915-USP410 datasheet.
- The stale `_autosave-Remote_*_Hand.kicad_sch` files should be deleted to stop them being
  mistaken for the real design.

---

## Revision log

Newest first. One row per change that altered the design.

> **This log begins on 2026-08-04.** Design work before that date predates the convention and
> was not recorded; the rows below start from when this document was written. Commit hashes are
> filled in once they exist.

| Date | Commit | Change | Why |
|---|---|---|---|
| 2026-08-10 | — | **U1 (AP2112K-3.3 LDO) removed; U2 (TPS63802 buck-boost) fitted**, with C42/C40/L1/R37/R39/R41 and the R13/R35/C33 undervoltage lockout. ERC clean | The LDO could not hold 3.3 V below ~3.6 V of cell, and peak load (~750–900 mA with the E22 transmitting) exceeded its ~600 mA rating. A buck-boost regulates across the whole Li-ion range, and its precise EN threshold buys cell protection for three passives |
| 2026-08-10 | — | **`VOUT` retyped `power_out`** in the `01-Custom` symbol | SnapEDA's generated symbol types VOUT as a plain `output`, which ERC does not treat as a power source. With the LDO gone, `3.3V_REG` read as undriven and every `power_in` on the rail errored. A `PWR_FLAG` would have masked it while leaving the symbol wrong for every future board |
| 2026-08-10 | — | Recorded that the package has **no separate thermal pad** — pad 8 (GND) is enlarged to 1.3 mm and carries the heat | Initially assumed a separate exposed pad from TI's land-pattern drawing; "EXPOSED METAL SHOWN" there describes solder-mask definition on the signal pads, not a thermal pad. The footprint has exactly ten pads. Recording it so the symbol isn't "corrected" later by adding a pin that has no pad |
| 2026-08-10 | — | **EN2 moved from `3.3V_REG` to `CHARGER_OUT`** | `3.3V_REG` dies with the power switch, collapsing EN2 to 0 → USB100 mode → **100 mA charging whenever the remote was switched off**, which is how a handheld is normally charged. `CHARGER_OUT` is live whenever USB or battery is present |
| 2026-08-10 | — | **R9 (ITERM) 17.8 kΩ → 3.0 kΩ** | Termination fired at 297 mA against a 494 mA fast charge — 60 % of charge current, versus TI's own example of ~14 %. The cell would never approach full. 3.0 kΩ gives 50 mA |
| 2026-08-10 | — | **C15 (0.1 µF on TMR) deleted; TMR left unconnected with an NC flag** | TMR accepts a resistor (18–72 kΩ), a tie to VSS, or nothing. A capacitor is not a valid configuration. Unconnected gives 30 min / 5 h defaults, ample against a ~2–2.5 h charge, and the timers stretch automatically under DPPM |
| 2026-08-10 | — | **R13 deleted — TS no longer fed from the battery**; TS now sits on R14 (10 kΩ) to GND alone | The 10 k/10 k divider from `JST_BAT+` put TS at ≈2.1 V on a full cell — **exactly V_COLD (2100 mV)** — so the charger read "battery too cold" and would inhibit charging intermittently as the cell drifted. It also drew 210 µA continuously. 10 kΩ to ground is TI's documented no-thermistor configuration and gives 750 mV, mid-band |
| 2026-08-10 | — | **Battery sense added**: `VBAT_SENSE` (470 k/470 k + 1 µF) → IO9; `BTN_1` moved IO9 → IO35; `CHG_STAT` → IO36, `PWR_GOOD` → IO37 | Nothing could read battery state. All twenty ADC-capable pins were occupied, so an ADC1 channel had to be freed by relocating a button. 470 kΩ rather than 10 kΩ keeps standby drain at ~4.5 µA instead of 210 µA |
| 2026-08-04 | — | Document created. Recorded the one-sheet-twice hierarchy, the verified pin map, the power chain, and the battery-monitoring gap | The repo had no design doc; the ADC exhaustion in §6 is a hard constraint that would otherwise be rediscovered at the point of adding the feature |
