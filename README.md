# AntAIO

Open-source **antweight / beetleweight combat-robot all-in-one** controller — an
ExpressLRS receiver, a 6-axis IMU, and a **triple brushed-DC ESC** on one tiny
board, in the spirit of the [Malenki Nano](https://turnabot.com/products/malenki-nano-integrated-6-channel-triple-electronic-speed-controller-receiver-combo). Built on
the **OpenRX-Lite** ELRS core (ESP32-C3 + SX1281, 2.4 GHz) and forked from
[`incutec-hw/OpenRX`](https://github.com/incutec-hw/OpenRX).

<p align="center">
  <img src="images/antaio-front.png" width="300" alt="AntAIO front" />
  <img src="images/antaio-back.png" width="300" alt="AntAIO back" />
</p>

> [!WARNING]
> **Prototype — not validated, not fab-ready.** This board has open
> safety-critical issues (uncommanded drive-motor motion at power-up, and the
> programming interface shares pins with the motor outputs). It drives combat-robot
> motors **including a weapon** — do not build or power it without reading the
> review first. See **[OpenRX-Lite/REVIEW.md](OpenRX-Lite/REVIEW.md)** (full design
> review) and **[CHANGELIST.md](CHANGELIST.md)** (verified to-do list).

## What it is

A single **10.00 × 21.50 mm**, 6-layer board that replaces the usual *RX + 2 drive ESCs +
weapon ESC + gyro* stack of an antweight robot with one MCU-driven combo:

- **Radio link** — ExpressLRS 2.4 GHz (Semtech SX1281) for RC control + telemetry,
  plus the ESP32-C3's own Wi-Fi for OTA config.
- **Motion sensing** — LSM6DS3TR-C 6-axis IMU (gyro + accel) for heading-hold /
  self-right / melty-style control in firmware.
- **Three brushed-DC motor channels** — two drive + one weapon, each a TI DRV8212
  H-bridge fed directly from the battery.
- **Single-cell-friendly power** — buck-boost holds a 5 V rail as the pack sags
  under weapon load, with a linear 3.3 V logic rail and reverse-polarity protection.

Everything is controlled by the **ESP32-C3**: RC decode, mixing, gyro stabilization,
and the three motor PWM outputs all run on-chip.

## Architecture

```
            +BATT (1–2S LiPo)
              │
        Q2  AON7407  ── reverse-polarity P-FET
              ├─────────────► VM ─► DRV8212 ×3 ─► LEFT / RIGHT drive + WEAPON motors
              │
        U8  TPS63070 buck-boost ─► +5V ─► U4 TLV75533 LDO ─► +3V3 (logic)
                                                              │
   ┌──────────────────────────────────────────────────────────┤
   │                                                            │
 U1 ESP32-C3 ──SPI──► U3 SX1281 (2.4 GHz ELRS) ─► FL1 BPF ─► link antenna
   │            └────► U2 LSM6DS3TR-C IMU                 ESP Wi-Fi ─► AE1
   ├── 3× PWM ─► DRV8212 IN/EN  (U6 LEFT, U10 RIGHT, U11 WEAPON)
   ├── ADC ──── VSENSE (battery voltage divider)
   └── GPIO ──► WS2812B status LED (D1)
```

## Specifications

| Block | Part | Notes |
|---|---|---|
| **MCU** | Espressif **ESP32-C3FH4** (U1, QFN-32) | RISC-V, 4 MB embedded flash, 2.4 GHz Wi-Fi |
| **Link radio** | Semtech **SX1281** (U3) | 2.4 GHz, internal DC-DC, 52 MHz TCXO (OSC1), BPF FL1 (2450FM07D0034T) → link antenna |
| **IMU** | **LSM6DS3TR-C** (U2) | 6-axis accel+gyro, 4-wire SPI on the shared radio bus, dedicated CS |
| **Motor drivers** | 3× TI **DRV8212DSGR** | U6 = LEFT drive, U10 = RIGHT drive, U11 = WEAPON; PH/EN mode; VM = +BATT |
| **Buck-boost** | TI **TPS63070** (U8) | +BATT → 5 V; holds the rail across the 1S–2S sag range |
| **LDO** | **TLV75533PDQNR** (U4) | 5 V → 3.3 V logic rail |
| **Reverse protection** | **AON7407** P-FET (Q2) | on +BATT |
| **Status LED** | WS2812B (D1) | addressable RGB |
| **Battery** | **1–2S LiPo (≤ 8.4 V)** | DRV8212 VM is 12 V abs-max — **2S is the hard ceiling; 3S destroys the drivers** |
| **Board** | 10.00 × 21.50 mm, 6-layer | components both sides |

## I/O pads

Solder pads carry the external interface (silk in parentheses):

| Pad | Net | Function |
|---|---|---|
| `+` / `–` | `+BATT` / `GND` | Battery in (1–2S) |
| `5U` | `+5V` | 5 V buck-boost output (servo / accessory) |
| `GND` | `GND` | Ground |
| `S` | GPIO | Signal / spare control pad |
| `BOOT` | GPIO 9 | Hold low at power-up for UART download mode |
| `R` / `T` | UART0 RX / TX | Flashing & serial console (also the right-drive lines — flash with motor battery OFF) |

Motor outputs (OUT1/OUT2 of each DRV8212) and the weapon channel land on the large
copper pads along the board edge.

## ESP32-C3 pin map (as currently drawn)

| GPIO | Pin | Function | GPIO | Pin | Function |
|---|---|---|---|---|---|
| 0 | 4 | VSENSE (battery ADC) | 8 | 14 | WS2812 LED *(strap)* |
| 1 | 5 | Radio DIO1 | 9 | 15 | BOOT *(strap)* |
| 2 | 6 | IMU CS *(strap)* | 10 | 16 | WEAPON_EN |
| 3 | 8 | Radio BUSY | 18 | 25 | LEFT_PH *(USB D−)* |
| 4 | 9 | SPI MOSI | 19 | 26 | LEFT_EN *(USB D+)* |
| 5 | 10 | SPI MISO | U0RXD | 27 | RIGHT_PH |
| 6 | 12 | SPI SCK | U0TXD | 28 | RIGHT_EN |
| 7 | 13 | Radio NSS | GND/EP | 33 | GND |

> [!IMPORTANT]
> The drive-motor enables currently sit on the USB (GPIO18/19) and UART0
> (U0RXD/U0TXD) pins, which are driven **high at reset** — enabling both drive
> motors before firmware runs. The planned fix re-maps PH↔EN onto reset-quiet pins
> and gates the H-bridges behind a default-off arm switch. **Do not treat this pin
> map as final** — track it in [CHANGELIST.md](CHANGELIST.md).

## Flashing

- **UART:** hold `BOOT` low, power-cycle, flash over the `R`/`T` (U0RXD/U0TXD) pads
  **with the motor battery disconnected** (these pins also drive the right motor).
- **Wi-Fi OTA:** ExpressLRS SoftAP, after an initial wired flash.

The ELRS link runs upstream **ExpressLRS** (`Unified_ESP32C3_2400_RX`, ≥ 3.5.0).
The combat-robot mixing, IMU stabilization, and motor-PWM control are application
firmware layered on top (separate; not in this hardware repo).

## Repository

This is a fork of `incutec-hw/OpenRX`; the upstream four-variant ELRS receiver line
is retained, and **OpenRX-Lite** is the board extended into AntAIO.

```
AntAIO/
├── OpenRX-Lite/          AntAIO board (this project): ELRS RX + IMU + triple ESC
│   ├── OpenRX-Lite.kicad_{pro,sch,pcb}
│   ├── esp32c3_sx1281_lite.kicad_sch
│   ├── REVIEW.md         full design review (pin/datasheet/electrical)
│   └── DESIGN.md
├── OpenRX-Lite-UFL/ · OpenRX-Mono/ · OpenRX-Gemini/   upstream OpenRX RX variants
├── shared/libs/          project-local KiCad symbols / footprints / 3D models
├── datasheets/common/    local datasheet cache
├── images/               board renders
└── CHANGELIST.md         verified open / done items for the AntAIO conversion
```

KiCad 9/10; **project-local libraries only** (no global dependencies). Board renders
are produced with the shared `render_board.py` tool (vias + solder paste stripped,
transparent square).

## License

Hardware: **CERN Open Hardware Licence Version 2 — Strongly Reciprocal**
([CERN-OHL-S-2.0](https://ohwr.org/cern_ohl_s_v2.txt)). See [LICENSE](LICENSE).
Firmware: MIT.
