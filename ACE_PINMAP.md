# ACE Pro — MCU Pin Map (GD32F303CCT6)

> **⚠️ UNVERIFIED **
>
> This pin map is reverse-engineered. Pin assignments may be
> incomplete or incorrect. **Verify with a multimeter/scope before relying on
> any of this for your own configuration.**

---

## Stepper Control (per slot)

The ACE Pro has 4 filament slots. The firmware uses indexed GPIO arrays.

| Slot | STEP pin | DIR pin | ENABLE / CLK pin |
|------|----------|---------|------------------|
| 0 | `PC9` | `PB12` | `PB13` |
| 1 | `PA8` | `PC6` | `PB13` |
| 2 | `PC9` | `PB12` | `PB13` |
| 3 | `PA8` | `PC6` | `PB13` |

Slots 0/2 share the same STEP/DIR lines; slots 1/3 share theirs.
`PB13` is shared across all slots.

### Per-slot switch/control lines

| Slot | GPIO |
|------|------|
| 0 | `PB3` |
| 1 | `PD2` |
| 2 | `PB5` |
| 3 | `PB4` |

---

## NFC Readers

Two MFRC522-style readers, each covering two slots.

| Reader | UART | Slots |
|--------|------|-------|
| 0 | `USART0` (PA9/PA10) | 0, 1 |
| 1 | `USART1` (PA2/PA3) | 2, 3 |

### NFC SPI bit-bang lines (shared)

| Signal | GPIO |
|--------|------|
| CLK | `PB13` |
| MOSI | `PB15` |
| MISO | `PB14` |

---

## Fan / Dryer / Thermal

| Function | Peripheral | GPIO |
|----------|-----------|------|
| Dryer fan PWM (speed) | TIM2 CH0 + CH1 | `PA6`, `PA7` |
| Secondary fan/control | TIM1 CH0 | (TBD) |
| PTC heater sense | GPIO input | `PA15` |
| Thermal sensor I²C bit-bang | GPIO | `PA15`, `PC10` |
| NTC/PTC ADC | ADC0 | (channel TBD) |

---

## USB

| Function | GPIO |
|----------|------|
| USB D− | `PA11` |
| USB D+ | `PA12` |
| D+ pullup enable (external transistor + 1.5 kΩ) | `PB9` HIGH = connect |

---

## Other

| Function | GPIO / Peripheral |
|----------|-------------------|
| RS485 direction (DE) | `PB9` (shared with USB pullup) |

---

## Confidence Levels

| Area | Confidence | Notes |
|------|-----------|-------|
| Stepper STEP/DIR per slot | Medium-High | From indexed array assignments in decompiled code |
| NFC USART mapping | High | Explicit init functions in firmware |
| NFC SPI bit-bang (PB13/14/15) | High | Direct read/write/toggle in `spi_bitbang_*` |
| Fan PWM pins (PA6/PA7) | Medium-High | Timer AF config in init path |
| PTC/thermal pins | Medium | Abstracted through state machine helpers |
| Heater drive pin | Low | Not directly identified — hidden behind PID/dryer abstraction |
