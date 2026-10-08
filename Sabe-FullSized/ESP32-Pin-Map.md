# Sabé (Full-Sized) — ESP32 Pin Map

Two ESP32 boards on the Sabé main board, generated from `Sabe-FullSized.kicad_sch`.

- **U5** — Espressif ESP32-S3-DEVKITC-1-N8R2 (main brain)
- **U7** — Seeed XIAO ESP32-C3 (co-processor / programmer bridge)

Only connected pins are listed; everything else is left as an unconnected pad.

---

## U5 — ESP32-S3-DEVKITC-1-N8R2

The DevKitC breaks out to two header rows, **J1** and **J3**.

### J1 header

| Hdr pin | GPIO | Net | Goes to |
|---------|------|-----|---------|
| J1-3  | EN/RST | `RST`      | U7 pin 2 (XIAO D1) |
| J1-4  | GPIO4  | `xbee_dout`| U4.2 (XBee) |
| J1-5  | GPIO5  | `xbee_din` | U4.3 (XBee) |
| J1-6  | GPIO6  | `xbee_rst` | U4.5 + R1 (XBee reset) |
| J1-7  | GPIO7  | `RBTQ_1`   | J1.3 |
| J1-8  | GPIO15 | `S3_RX`    | J4.3 (serial port 3) |
| J1-9  | GPIO16 | `S3_TX`    | J4.4 (serial port 3) |
| J1-12 | GPIO8  | `RBTQ_2`   | J1.4 |
| J1-15 | GPIO9  | `Dome_PWM` | J6.2 |
| J1-16 | GPIO10 | `ESTOP`    | J11.2 |
| J1-17 | GPIO11 | `S1_RX`    | J2.4 (serial port 1) |
| J1-18 | GPIO12 | `S1_TX`    | J2.3 (serial port 1) |
| J1-19 | GPIO13 | `S2_RX`    | J3.4 (serial port 2) |
| J1-20 | GPIO14 | `S2_TX`    | J3.3 (serial port 2) |
| J1-21 | 5V     | `5V`       | 5V rail (also feeds U7-14) |
| J1-1/2| 3V3    | —          | tied together, otherwise unused |
| J1-22 | GND    | `GND`      | GND |

### J3 header

| Hdr pin | GPIO | Net | Goes to |
|---------|------|-----|---------|
| J3-2  | GPIO43 (U0TXD) | `S6TX_C6RX` | R2 → U7 pin 9 (XIAO D8) |
| J3-3  | GPIO44 (U0RXD) | `S6RX_C6TX` | R3 → U7 pin 10 (XIAO D9) |
| J3-4  | GPIO1  | `Misc1_G1`  | J7.3 |
| J3-5  | GPIO2  | `Misc1_G2`  | J7.4 |
| J3-14 | GPIO0  | `GPIO_0`    | U7 pin 3 (XIAO D2) |
| J3-16 | GPIO48 | `Misc3_G48` | J9.4 |
| J3-17 | GPIO47 | `Misc3_G47` | J9.3 |
| J3-1/21/22 | GND | `GND`    | GND |

---

## U7 — Seeed XIAO ESP32-C3

The GPIO column is the standard XIAO ESP32-C3 silk → GPIO mapping.

| XIAO pin | Silk / GPIO | Net | Goes to |
|----------|-------------|-----|---------|
| 1  | D0 / GPIO2     | —              | unconnected |
| 2  | D1 / GPIO3     | `RST`          | **U5 EN/RST** (J1-3) |
| 3  | D2 / GPIO4     | `GPIO_0`       | **U5 GPIO0** (J3-14) |
| 4  | D3 / GPIO5     | —              | unconnected |
| 5  | D4 / GPIO6     | —              | unconnected |
| 6  | D5 / GPIO7     | —              | unconnected |
| 7  | D6/TX / GPIO21 | —              | unconnected |
| 8  | D7/RX / GPIO20 | —              | unconnected |
| 9  | D8 / GPIO8     | `Net-(U7-D8)`  | R2 → **U5 GPIO43 / U0TXD** |
| 10 | D9 / GPIO9     | `Net-(U7-D9)`  | R3 → **U5 GPIO44 / U0RXD** |
| 11 | D10 / GPIO10   | —              | unconnected |
| 12 | 3V3            | —              | unconnected (floating) |
| 13 | GND            | `GND`          | GND |
| 14 | 5V (VUSB)      | `5V`           | 5V rail (with U5 J1-21) |

---

## How the two ESP32s interconnect

The XIAO C3 is wired as a **USB-serial programmer / bridge for the S3** — four
signals plus shared power:

| Function    | XIAO C3 (U7)   | ESP32-S3 (U5)     | Note |
|-------------|----------------|-------------------|------|
| Reset       | D1 / GPIO3     | EN/RST            | C3 can reset the S3 |
| Boot strap  | D2 / GPIO4     | GPIO0             | RST + IO0 = auto-enter download mode |
| UART        | D8 / GPIO8     | GPIO43 (U0TXD)    | via R2 series resistor |
| UART        | D9 / GPIO9     | GPIO44 (U0RXD)    | via R3 series resistor |

The C3 drives EN + IO0 (the classic auto-program handshake) and bridges the S3's
UART0 through series resistors — i.e. the C3 flashes/monitors the S3. The net
names `S6TX_C6RX` / `S6RX_C6TX` on the S3 side confirm the intended
S3-UART0 ↔ C3 link. Both boards share the 5V rail; the C3's 3V3 pin is left
floating.

> Note: the C3's native USB-visible UART (D6/D7, GPIO20/21) is unconnected, so
> the bridge uses D8/D9 (GPIO8/9) — a software UART assignment on the C3, not the
> hardware USB-CDC pins.
