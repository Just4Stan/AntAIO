# Design Notes

Human-readable mirror of the on-canvas comments annotated directly on the OpenRX
KiCad schematics (`*.kicad_sch`). Quoted lines are reproduced **verbatim** from
the sheets; surrounding context (which reference designator / part implements a
block) is derived from the schematic symbols and net labels. Keep this file in
sync with the on-canvas annotations.

The four variants share one ESP32-C3 core; they differ in radio and RF front-end.
Notes are grouped by the shared core first, then per-variant radio sections.

---

## Shared ESP32-C3 core

Present on every variant — `esp32c3_sx1281_lite.kicad_sch` (Lite / Lite-UFL),
`esp32c3_lr1121_mono.kicad_sch` (Mono), and `esp32-c3.kicad_sch` (Gemini).

On-canvas block labels: **“ESP32C3 MCU”**, **“LDO”**.

### LDO (U2 — TLV75533PDQNR)
- *“5V -> 3.3V, 238mV@500mA maximum dropout”* — 5 V from the `5V` pad to the
  3.3 V rail; worst-case dropout 238 mV at 500 mA.

### ESP32-C3 (U1) GPIO groupings
GPIO clusters annotated on the sheet near the MCU / pad area:
- **“GPIO0 GPIO1”**
- **“GPIO4 GPIO5 GPIO6 GPIO7”** — radio SPI/control group.
- **“GPIO21 GPIO20”** — UART0 TX / RX, broken out to the `TX` / `RX` pads.

GPIO 9 is the BOOT/strapping pin (the `BOOT` pad on Lite/Lite-UFL/Mono; a tactile
button, U9, on Gemini). The WS2812 RGB status LED (D1) is driven from the MCU and
powered from +3.3 V.

---

## Lite / Lite-UFL — SX1281 2.4 GHz (`esp32c3_sx1281_lite.kicad_sch`)

Sheet title: **“SX1281 + ESP32-C3 + LDO + RGB LED (full ELRS)”**.
Radio block: **“SX1281 radio”**.

- Single Semtech **SX1281** (U3) 2.4 GHz transceiver.
- RF path: SX1281 RFIO → **2450FM07D0034T** band-pass filter (FL1) → ELRS antenna.
  - **Lite:** antenna is the on-board **47948-0001** chip antenna (AE2).
  - **Lite-UFL:** antenna is the **U.FL** connector (J1).
- The **2450AT18A100E** ceramic chip antenna (AE1) is the ESP32-C3 **Wi-Fi**
  antenna (net `WIFI`), not the ELRS antenna.
- No RF front-end (no PA/LNA, no RF switch, no sub-GHz).

---

## Mono — single LR1121 dual-band (`esp32c3_lr1121_mono.kicad_sch`)

Sheet labels: **“Mono ELRS (2.4+900)”**, **“LR1121 Radio”**.

- Single Semtech **LR1121** (U3) — multi-band (sub-GHz + 2.4 GHz).
- RF front-end: **RFX2401C** PA/LNA (U4) + **SKY13373-460LF** RF switch (U5) +
  **0900PC16J0042001E** balun/IPD (T1) + **2450FM07D0034T** BPF (FL1) → **U.FL** (J1).
- 32 MHz TCXO (OSC1, OW7EL89CENUYO3YLC-32M).

---

## Gemini — dual LR1121 Xrossband (`OpenRX-Gemini.kicad_sch`)

Top sheet note: **“2 instances of LR1121 transceiver, ESP32-C3, clock. Gemeni ELRS”**
(hierarchical: `esp32-c3.kicad_sch` + `clock.kicad_sch` + `lr1121.kicad_sch` ×2).

### ESP32-C3 sheet
- **“ESP32-C3 + peripherals”** — shared core (same MCU/LDO/LED as above) plus the
  tactile BOOT button (U9, GPIO 9).

### Clock sheet (`clock.kicad_sch`)
- **“clock”** — single 32 MHz TCXO (OSC1) shared by both radios.

### Radio sheet (`lr1121.kicad_sch`, instantiated ×2)
- **“LR1121 Radio”**, **“LR1121 + 2.4/900 matching + LNA/PA + RF switch”**.
- Each instance: LR1121 (U3) + RFX2401C PA/LNA (U4) + SKY13373-460LF switch (U5)
  + 0900PC16J0042001E balun (T1) + 2450FM07D0034T BPF (FL1) → its own **U.FL** (J1/J2).
- Two complete radio chains enable dual-band / Xrossband diversity.
