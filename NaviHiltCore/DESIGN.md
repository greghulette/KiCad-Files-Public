# NaviHiltCore — Design Notes

Hilt companion board for the R2 handheld (HH) controllers. An ESP32-S3 DevKitC running
[NaviCore](https://github.com/greghulette/NaviCore) firmware, with **two** EBYTE E22-900M22S
915MHz LoRa transceivers — one per remote — plus SBUS in/out, three PWM outputs, a Maestro
port, and two general serial ports.

Files: `NaviHiltCore.kicad_sch` / `.kicad_pcb`, KiCad 10. Wired with local net labels rather
than drawn wires.

Read [../docs/LIBRARIES.md](../docs/LIBRARIES.md) and
[../docs/KICAD_EDITING.md](../docs/KICAD_EDITING.md) before editing the schematic.

---

## 1. The MCU and why it constrains everything

**U1 = ESP32-S3-DevKitC-1-N16R8.**

- The **N16R8 has octal PSRAM, and octal PSRAM consumes GPIO33–37.** Those five pins are
  physically unusable. NaviCore firmware *requires* the PSRAM, so this is not negotiable —
  and it is the single most common way to design a broken pinout for this board. An earlier
  draft put E22 signals there.
- The devkit (not a bare module) is deliberate: it brings native USB (the CDC port NaviCore's
  config tool connects to), the EN/boot circuit, and an onboard status NeoPixel on GPIO48. No
  external USB, boot, or LED parts are needed.
- The symbol is `01-Custom:ESP32-S3-DEVKITC-1-N8R2` even though the part is the N16R8. The
  header layout is identical across DevKitC-1 variants, so the symbol is correct; the
  *instance* Value carries the real part. GPIO35/36/37 are left unconnected, which is right
  for octal PSRAM.
- Footprint `01-Custom:ESP32-S3DevKit-Multi`, pads numbered `J1_x` / `J3_x` for the two
  header rows.

### The pin budget

After NaviCore's own needs — native USB (19, 20), strapping pins (0, 3, 45, 46), the UART0
bridge (43, 44), the status LED (48) — and PSRAM's 33–37, roughly **14 clean GPIOs remain**.

Two fully independent bidirectional E22s need 15 pins on their own, before any PWM. Three
things make it fit:

1. **The two E22s share one SPI bus** (MOSI/SCK/MISO) and **one reset line**. Per-module soft
   reset happens over SPI instead.
2. **Serial3 and PWM4 were dropped.** Their connectors (J5, J9) are removed from the
   schematic, not just unpopulated.
3. **PWM1 lives on GPIO43 (U0TXD)** — see the warning in §2.

## 2. Pin map

Verified against `NaviHiltCore.kicad_sch`.

### Shared / system

| GPIO | Header pad | Net | Notes |
|---|---|---|---|
| 4  | J1_4  | `SBUS_IN`  | NaviCore's stock SBUS RX — J11, from an FrSky receiver |
| 5  | J1_5  | `SBUS_OUT` | J1 |
| 48 | —     | —          | onboard status NeoPixel, no external wiring |
| —  | J1_21 | `+5V`      | the DevKitC is fed 5V and makes its own 3V3 |

### Peripherals

| GPIO | Header pad | Net | Connector |
|---|---|---|---|
| 6  | J1_6  | `MAESTRO_TX` | J6 (Maestro, 4-pin) |
| 7  | J1_7  | `MAESTRO_RX` | J6 |
| 8  | J1_12 | `S1_TX` | J7 (Serial1, 4-pin) |
| 9  | J1_15 | `S1_RX` | J7 |
| 10 | J1_16 | `S2_TX` | J8 (Serial2, 4-pin) |
| 21 | J3_18 | `S2_RX` | J8 |
| 43 | J3_2  | `PWM1` | J2 — **U0TXD**, see below |
| 2  | J3_5  | `PWM2` | J3 |
| 1  | J3_4  | `PWM3` | J4 |

> **PWM1 sits on GPIO43 = U0TXD.** NaviCore talks to the config tool over *native* USB CDC,
> so UART0 is free and this works. But anything that prints to a UART0-backed `Serial` will
> drive PWM1 — a servo twitching on boot or during debug output is this, not a wiring fault.

### E22 LoRa transceivers

Shared bus: **MOSI = 11, SCK = 12, MISO = 13, RST = 47** (one reset net for both).

| Signal | IC1 | IC2 |
|---|---|---|
| NSS  | 14 (J1_20) | 38 (J3_10) |
| BUSY | 15 (J1_8)  | 39 (J3_9)  |
| DIO1 | 16 (J1_9)  | 40 (J3_8)  |
| RXEN | 17 (J1_10) | 41 (J3_7)  |
| TXEN | 18 (J1_11) | 42 (J3_6)  |

Both modules are **bidirectional**, one per remote.

### Spare

**GPIO44** (U0RXD) and **GPIO3**. Both carry `no_connect` flags — delete the flag to use one.

## 3. Power

`J10` (2-pin screw terminal) takes **+5V and GND** in.

Two rails:

- **DevKitC 3V3** — the devkit's own AP2112, ~600mA. Not enough for the radios.
- **`+3V3_RF`** — **U2 = Pololu D24V22F3** (#2857, 3.3V 2.6A buck), feeding both E22s only.

Each E22 draws ~500–650mA while transmitting; worst case is ~1.3A. That is why the radios get
their own regulator rather than hanging off the devkit.

U2 wiring: VIN and EN ← `+5V`, VOUT → `+3V3_RF`, GND → `GND`, PG → no-connect.
Custom symbol and footprint `01-Custom:D24V22F3`. **The footprint omits the two M2 mounting
holes** (13.21mm apart per Pololu's dimension drawing) — add them at layout.

Decoupling C1–C6: 100µF + 10µF + 100nF per E22.

PWR_FLAGs `#FLG01`–`#FLG03` on `+5V`, `GND`, `+3V3`. **`+3V3_RF` deliberately has no
PWR_FLAG** — U2's VOUT is typed `power_out` and already drives the rail; adding a flag makes
two power sources and an ERC conflict.

## 4. RF

Each E22's ANT pin goes to its own SMA connector — IC1 → `ANT1` → **J12**, IC2 → `ANT2` →
**J13**, shields to GND. Symbol `Connector:Conn_Coaxial`, footprint
`Connector_Coaxial:SMA_Amphenol_132134_Vertical`.

Antennas: Linx ANT-915-NUB-SMA (SMA male), confirmed suitable for both 915MHz modules.

**Layout requirements** — these are the ones that decide whether the radios work:

- Short, 50Ω controlled-impedance traces from each E22's ANT pad to its SMA.
- Ground pour with stitching vias around the feeds.
- Separate the two feeds, and the two antennas, as far as the enclosure allows.

## 5. Connectors

| Ref | Function | Pins | Order |
|---|---|---|---|
| J1  | SBUS Out | 3 | Signal / +5V / GND |
| J2/J3/J4 | PWM1 / PWM2 / PWM3 | 3 | Signal / +5V / GND |
| J6  | Maestro  | 4 | GND / +5V / TX / RX |
| J7/J8 | Serial1 / Serial2 | 4 | GND / +5V / TX / RX |
| J10 | Power in | 2 | +5V (1) / GND (2) |
| J11 | SBUS In  | 3 | Signal / +5V / GND |
| J12/J13 | SMA antennas | — | IC1 / IC2 |

3- and 4-pin headers are vertical (`PinHeader_1x0N_P2.54mm_Vertical`).

## 6. Status and open items

ERC: **0 errors, 0 warnings.**

Open:

- **U2's mounting holes** are not in the footprint — add at layout (§3).
- **SMA footprint** is the vertical Amphenol part; confirm or swap for edge-mount /
  right-angle once the enclosure is settled.
- `fp-lib-table` in this directory points at a **nonexistent `D:/` path** and is unused —
  see [../docs/LIBRARIES.md §4](../docs/LIBRARIES.md#4-known-broken-entries). Safe to delete.
- PCB layout is not finished; the RF constraints in §4 are the priority.

---

## Revision log

Newest first. One row per change that altered the design.

> **Rows dated before 2026-08-04 are reconstructed**, not contemporaneous. They were recovered
> on 2026-08-04 from working notes kept during the design, and are dated to the day rather
> than the hour. Commit hashes are unavailable for them because the schematic edits were not
> committed individually. Everything from 2026-08-04 onward is logged as it happens.

| Date | Commit | Change | Why |
|---|---|---|---|
| 2026-08-04 | — | Design notes moved here from a session memory file; pin map re-verified against `NaviHiltCore.kicad_sch` | Notes had accumulated superseded sections and weren't version controlled |
| 2026-07-24 | — | `Footprint` field on IC1/IC2 corrected to `01-Custom:E22-900M22S` | Was a bogus nickname (`E22-900M22S:E22-900M22S`), which presents exactly like a missing library |
| 2026-07-24 | — | U1 instance Value reconciled to `ESP32-S3-DevKitC-1-N16R8` | Symbol is the N8R2 variant; the real part is N16R8 and the instance should say so |
| 2026-07-24 | — | E22 ANT pins wired to SMA connectors J12 / J13 | u.FL was inadequate for the enclosure; SMA lets the antennas mount externally |
| 2026-07-24 | — | SBUS-IN added on GPIO4 with connector J11; **PWM1 moved to GPIO43** | SBUS turned out to need a real input after all, so GPIO4 couldn't stay reused for PWM |
| 2026-07-24 | — | **U2 corrected to Pololu D24V22F3** (#2857, 3.3V 2.6A); custom symbol + footprint built | The previously specified `D24V50F3` **does not exist** — Pololu's D24V50 family is 5V+ — and the footprint borrowed from the 5V board had the wrong pin order. Caught at the point of ordering |
| 2026-07-24 | — | `+3V3_RF` PWR_FLAG removed | U2's VOUT is typed `power_out` and drives the rail; two power sources is an ERC conflict |
| 2026-07-23 | — | **U1 symbol swapped** to `01-Custom:ESP32-S3-DEVKITC-1-N8R2`; all 27 net labels and 13 no-connects re-mapped; 5V pin (J1_21) wired | "Update PCB" failed outright — U1 used a bare-module symbol (pins 1–65) against a devkit footprint (pads J1_x/J3_x). Total pad mismatch |
| 2026-07-23 | — | Connectors J5 (PWM4) and J9 (Serial3) removed from the schematic | Pin budget: two bidirectional E22s plus PWM didn't fit in the ~14 clean GPIOs left |
| 2026-07-23 | — | E22 reset lines merged to one shared `RST` net | Same pin-budget squeeze; per-module reset happens over SPI instead |
| 2026-07-23 | — | Radios moved onto their own `+3V3_RF` rail from a dedicated buck | The DevKitC's onboard AP2112 supplies ~600mA; two E22s peak around 1.3A |
| 2026-07-23 | — | E22 signals moved off GPIO33–37 | **Octal PSRAM on the N16R8 consumes those pins.** An early draft had them assigned |
