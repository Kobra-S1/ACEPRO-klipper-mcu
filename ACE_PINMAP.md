# ACE Pro — MCU Pin Map (GD32F303CCT6)

> **Reverse-engineered from the ACE stock firmware**
>
> This file reflects the current hardware io usage based on firmware analysis and GD32F303 datasheet cross-check.

---

## Motor Control

The firmware model is consistent with **one selector motor** and **one
feed/retract motor**, not four per-slot motors.

| Motor role | STEP pin | DIR pin | Evidence |
|---|---|---|---|
| Selector / clutch-position motor | `PD2` | `PB3` | Channel 0 in `motor_gpio_table_init()`. Used by `gear_home()`, `gear_move_to_pos()`, `gear_begin_feed()` on `g_gear_a_ctx` |
| Filament feed / retract motor | `PB4` | `PB5` | Channel 1 in `motor_gpio_table_init()`. Used by `gear_exec_continuous_move()` / `gear_stop_and_reset()` on `g_gear_b_ctx` |

### Shared motor control

| Function | GPIO | Notes |
|---|---|---|
| Common control / enable / reset (polarity unclear) | `PA1` | Stored by `motor_gpio_table_init()`, toggled through `gpio_pin_write()` during init/retry paths |
| Driver-control outputs (exact role unresolved) | `PC1`, `PC2`, `PC3`, `PB0`, `PB1` | Configured as outputs in `motor_gpio_cfg()` before the TMC / motor bring-up path |

### What this means

- The selector motor rotates the slot-selection clutch / gear mechanism until
  the requested slot position is reached.
- After selection, the separate feed motor drives filament forward or backward
  through the chosen slot path.
- The exact external clutch linkage is still inferred from firmware behavior,
  but the selector-vs-feed split is strongly supported by the call graph.

---

## Slot Sensors

### Confirmed per-slot filament input pins

These are sampled directly in `rfid_cmd_init_register()` and line up with the
firmware's per-slot `"input"` diagnostic field.

| Slot | GPIO |
|---|---|
| 0 | `PA4` |
| 1 | `PA5` |
| 2 | `PC4` |
| 3 | `PC5` |

### Confirmed hall sensors

Factory status exposes:

- per-slot `input`
- per-slot `hall1`
- shared `hall2`

The slot sensor polling routine at `0x0800B250` debounces these GPIOs:

| Signal | GPIO | Notes |
|---|---|---|
| Slot 0 `hall1` | `PC13` | Debounced into `DAT_200000c0` |
| Slot 1 `hall1` | `PC14` | Debounced into `DAT_200000c4` |
| Slot 2 `hall1` | `PC15` | Debounced into `DAT_200000c8` |
| Slot 3 `hall1` | `PC0` | Debounced into `DAT_200000cc` |
| Shared `hall2` | `PB8` | Debounced into `DAT_200000d0`, reused for all four slots in factory status |

---

## NFC / RFID

The ACE Pro has **two NFC reader channels**. Each reader covers two slots.

| Reader | Peripheral | GPIOs | Slots |
|---|---|---|---|
| 0 | `USART0` | `PA9` / `PA10` | 0, 1 |
| 1 | `USART1` | `PA2` / `PA3` | 2, 3 |

### Per-slot NFC GPIO bundle

These come from the 4-slot table built by `nfc_gpio_table_init()`.

| Slot | RESET | IRQ | CS |
|---|---|---|---|
| 0 | `PC9` | `PC7` | `PB12` |
| 1 | `PA8` | `PC8` | `PC6` |
| 2 | `PC9` | `PC7` | `PB12` |
| 3 | `PA8` | `PC8` | `PC6` |

### Shared NFC bit-banged SPI lines

| Signal | GPIO |
|---|---|
| CLK | `PB13` |
| MOSI | `PB15` |
| MISO | `PB14` |

### Shared NFC control

| Function | GPIO | Notes |
|---|---|---|
| Common reader control line (enable/reset, polarity unclear) | `PA1` | Driven by `gpio_pin_write()` and initialized in `motor_channel_gpio_init()` |

---

## Fan / Dryer / Thermal

| Function | Peripheral | GPIO | Confidence |
|---|---|---|---|
| Dryer fan PWM | `TIMER2` CH0 + CH1 (`0x40000400`) | `PA6`, `PA7` | High |
| Auxiliary PWM output | `TIMER1` CH0 (`0x40000000`) | `PA0` | Medium-High |
| Status LED | GPIO output | `PB7` | High |
| Slot 0 indicator LED | GPIO output | `PB10` | High |
| Slot 1 indicator LED | GPIO output | `PB11` | High |
| Slot 2 indicator LED | GPIO output | `PA14` | High |
| Slot 3 indicator LED | GPIO output | `PA13` | High |
| PTC status input | GPIO input | `PA15` | High |
| Sensor bit-bang lines | GPIO | `PA15`, `PC10` | Medium |

### Notes

- `dryer_fan_pwm_init()` configures `PA6` and `PA7`, and
  `dryer_fan_pwm_set()` drives both `TIMER2` channels with the same duty.
- `aux_pwm_led_init()` configures `PA0` on the `TIMER1_CH0` path plus the
  `PB7` LED path.
- `slot_led_gpio_init()` configures `PB10`, `PB11`, `PA14`, and `PA13` as a
  four-output indicator group.
- `slot_led_write()` drives those four outputs both per-slot and as a 4-bit
  group mask. The observed write pattern suggests the slot LEDs are wired
  active-low, but the exact external polarity is still treated as an inference.
- `dryer_ptc_read()` reads `PA15` as a digital status input.
- The firmware definitely converts raw temperature readings through
  `adc_raw_to_float()`, but the exact end-to-end physical source of all
  temperature channels is still not fully resolved.

### Datasheet cross-checks

These mappings are consistent with `GD32F303xx_Datasheet_Rev3.1.pdf`:

- `PA6` = `TIMER2_CH0`, `PA7` = `TIMER2_CH1`
- `PA0` = `TIMER1_CH0`
- `PA9/PA10` = default `USART0_TX/RX`
- `PA2/PA3` = default `USART1_TX/RX`
- `PA11/PA12` expose `USBDM/USBDP`
- `PC6/PC7/PC8/PC9` are valid `TIMER2` remap pins
- `PB3/PB4/PB5` are valid GPIOs with alternate/remap roles available

---

## USB / Comms

| Function | GPIO | Confidence |
|---|---|---|
| USB D- | `PA11` | Medium |
| USB D+ | `PA12` | Medium |
| Comm / pull-up control output | `PB9` | Medium |
| Additional comm-related GPIO touched during init | `PB2` | Low |

`comm_gpio_init()` clearly configures `PB9` and `PB2`. In prior repo analysis,
`PB9` was correlated with the external USB D+ pull-up / RS485-DE style control
path, but the currently loaded app image alone does not fully disambiguate the
exact external function.

---

## Confidence Summary

| Area | Confidence | Notes |
|---|---|---|
| Selector motor (`PD2/PB3`) vs feed motor (`PB4/PB5`) split | High | Channel 0 is used by `gear_home()` / `gear_move_to_pos()`, channel 1 by continuous feed/retract routines |
| Four per-slot filament input pins (`PA4/PA5/PC4/PC5`) | High | Direct reads in `rfid_cmd_init_register()` |
| NFC reader USART mapping | High | Explicit init functions |
| NFC per-slot `RESET/IRQ/CS` table | High | Direct table initialization in `nfc_gpio_table_init()` |
| NFC shared `PB13/PB14/PB15` lines | High | Direct bit-banged read/write/clock helpers |
| `PA6/PA7` dryer PWM | High | Explicit timer AF setup and PWM writes |
| `PA0` auxiliary PWM | Medium-High | Explicit timer/PWM init and duty writes |
| `PB7` status LED | High | Direct set/clear helpers |
| Slot indicator LED GPIOs (`PB10/PB11/PA14/PA13`) | High | Direct per-slot and packed-mask writes |
| `hall1` / `hall2` physical pin mapping | High | Confirmed by the debouncing poller at `0x0800B250` |
| Exact role of `PA1` common control line | Medium | Clearly shared and toggled during init, exact external polarity/function still unresolved |

## Lower-Confidence / Unresolved

| Item | Confidence | Notes |
|---|---|---|
| Exact external clutch linkage | Medium | Selector positioning before feed is clear, but the full mechanical linkage is not directly visible in firmware |
| Exact role / polarity of `PA1` | Medium | Shared control line seen during init and retry paths |
| Exact electrical polarity of the four slot LEDs | Medium | The GPIO drive pattern suggests active-low wiring, but that has not been electrically verified |
| Full temperature input chain | Low-Medium | `PA15` digital status input is confirmed, but the full physical path for all thermal channels is not fully resolved |
| `PB9` external function | Medium | Configured in comm init; likely USB pull-up / RS485-style external control |
