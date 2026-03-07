# ACEPRO Klipper MCU — Patched Klipper for GD32F303

> **🚧 IMPORTANT — THIS IS NOT A FUNCTIONAL ACE PRO INTEGRATION**
>
> This repository **only** enables running Klipper firmware on the ACE Pro MCU
> hardware. It does **not** include any Klipper extras module, plugin, or
> configuration to actually operate the ACE Pro (filament slots, hub motor,
> sensors, etc.).
>
> **Without a dedicated Klipper extras module to drive it, the ACE Pro will be
> completely non-functional — effectively a very expensive brick.** No such
> module exists yet.
>
> **Only use this if you intend to develop the missing Klipper integration
> yourself.** If you just want your ACE Pro to work with Klipper, this repository
> is not what you are looking for.

---

> **⚠️ WARNING — READ BEFORE PROCEEDING**
>
> This repository contains patches and tools to run **Klipper** firmware on the
> **Anycubic ACE Pro** filament changer MCU (GD32F303CCT6).
>
> Flashing third-party firmware replaces the stock ACE application. If something
> goes wrong, your ACE Pro **may be bricked** and require an SWD debugger to
> recover. With Katapult installed (32 KiB offset), you can switch between
> Klipper and patched stock ACE firmware without SWD — see the
> [Katapult repo](https://github.com/Kobra-S1/ACEPRO-katapult-bootloader).
>
> The authors accept **no responsibility** for damaged, bricked, or non-functional
> devices. You use this entirely **at your own risk**.
>
> **If you do not know what Klipper, Katapult, or a bootloader is — this
> repository is not intended for you. Do not proceed.**

---

## Overview

The GD32F303CCT6 in the ACE Pro is register-compatible with STM32F103 but has
256 KiB flash / 48 KiB RAM and an **external D+ pullup on PB9** that upstream
Klipper doesn't know about. This repo provides:

- **`klipper-ace-gd32f303.patch`** — Patch for upstream Klipper (3 files: Kconfig,
  usbfs.c, flash_can.py) that adds GD32F303 support with working USB
- **Documentation** — Detailed technical reference for the USB fix, flash layout,
  and pin map

## Prerequisites

- **Katapult bootloader** already installed on the ACE Pro at 32 KiB offset
  (see [ACEPRO-katapult-bootloader](https://github.com/Kobra-S1/ACEPRO-katapult-bootloader))
- **Python 3.8+** with `pyserial`
- **ARM GCC toolchain**:
  ```bash
  # Ubuntu/Debian
  sudo apt install gcc-arm-none-eabi
  # Fedora
  sudo dnf install arm-none-eabi-gcc-cs arm-none-eabi-newlib
  ```

## Quick Start

### 1. Clone and patch Klipper

```bash
git clone https://github.com/Klipper3d/klipper.git
cd klipper
git apply /path/to/ACEPRO-klipper-mcu/klipper-ace-gd32f303.patch
```

### 2. Configure

```bash
make menuconfig
```

Select:
```
Micro-controller Architecture  → STMicroelectronics STM32
Processor model                → STM32F103
Low-level options              → [*]
  256KiB Flash / 48KiB RAM    → [*]
Bootloader offset              → 32KiB bootloader (Katapult)
Clock Reference                → 8 MHz crystal
Communication interface        → USB (on PA11/PA12)
```

### 3. Build

```bash
make -j$(nproc)
```

Output: `out/klipper.elf` and `out/klipper.bin`

### 4. Flash via Katapult

```bash
# Find the Katapult device
ls /dev/serial/by-id/ | grep katapult

# Flash
python3 /path/to/katapult/scripts/flash_can.py \
  -d /dev/serial/by-id/usb-katapult_stm32f103xe_*-if00 \
  -f out/klipper.bin
```

### 5. Verify

```bash
lsusb | grep 1d50:614e
# Expected: OpenMoko, Inc. stm32f103xe

ls /dev/serial/by-id/ | grep klipper
# Expected: usb-Klipper_stm32f103xe_...-if00
```

Expected `dmesg` output when flashing Klipper via Katapult (bus/device numbers will vary):

```
# Katapult bootloader waiting (1d50:6177)
usb 5-3.1.2: New USB device found, idVendor=1d50, idProduct=6177, bcdDevice= 1.00
usb 5-3.1.2: Product: stm32f103xe
usb 5-3.1.2: Manufacturer: katapult
usb 5-3.1.2: SerialNumber: 2BCF7AC5A461301239313538
cdc_acm 5-3.1.2:1.0: ttyACM2: USB ACM device

# flash_can.py flashes Klipper, device resets
usb 5-3.1.2: USB disconnect, device number 70

# ✓ Klipper is now running (1d50:614e)
usb 5-3.1.2: new full-speed USB device number 71 using xhci_hcd
usb 5-3.1.2: New USB device found, idVendor=1d50, idProduct=614e, bcdDevice= 1.00
usb 5-3.1.2: Product: stm32f103xe
usb 5-3.1.2: Manufacturer: Klipper
usb 5-3.1.2: SerialNumber: 2BCF7AC5A461301239313538
cdc_acm 5-3.1.2:1.0: ttyACM2: USB ACM device
```

## Updating Klipper Later

Katapult stays permanently in the first 32 KiB. To re-flash Klipper:

```bash
# Enter bootloader mode from running Klipper
python3 /path/to/katapult/scripts/flash_can.py \
  -d /dev/serial/by-id/usb-Klipper_stm32f103xe_*-if00 \
  -r

# Flash new firmware
python3 /path/to/katapult/scripts/flash_can.py \
  -d /dev/serial/by-id/usb-katapult_stm32f103xe_*-if00 \
  -f out/klipper.bin
```

## Switching Back to Stock ACE

With Katapult at 32 KiB, you can flash the stock ACE firmware back via
Katapult — see the "Switching to Stock ACE Firmware" section in the
[Katapult repo](https://github.com/Kobra-S1/ACEPRO-katapult-bootloader).

## What the Patch Fixes

| Problem | Root cause | Fix |
|---------|-----------|-----|
| No USB device detected | Board uses external D+ pullup on **PB9** — Klipper never drives it | Drive PB9 HIGH after USB init |
| USB init race condition | GD32F303 needs **tSTARTUP delay** between clearing PDWN and FRES | Insert `udelay(10)` between CNTR writes |
| Wrong flash/RAM size | GD32F303CC has 256K/48K, Klipper defaults to 64K/20K | Add `MACH_STM32F103xC` variant in Kconfig |
| `flash_can.py` crash on Python 3.14+ | Deprecated `asyncio.get_event_loop()` | Use `asyncio.new_event_loop()` |

## Memory Map

```
0x08000000 ┌──────────────────────┐
           │ Katapult (32 KiB)    │  ← USB bootloader (actual ~4 KiB,
           │                      │     reserves 32 KiB for app alignment)
0x08008000 ├──────────────────────┤
           │ Klipper (~40 KiB)    │  ← flashed via Katapult
           │   — or —             │
           │ Stock ACE (~103 KiB) │  ← flashed via Katapult (patched)
           │                      │
0x08040000 └──────────────────────┘
           256 KiB total flash
```

The 32 KiB offset is chosen so that both Klipper and the stock ACE firmware
share the same base address (`0x08008000`), enabling dual-boot via Katapult.

## File Overview

| File | Description |
|------|-------------|
| `klipper-ace-gd32f303.patch` | Patch for upstream Klipper — GD32F303 USB support (Kconfig, usbfs.c, flash_can.py) |
| `ace-pro-example.cfg` | Example Klipper config sections for the ACE Pro (⚠️ unverified — starting point only) |
| `ACE_PINMAP.md` | MCU pin map reverse-engineered from stock firmware (⚠️ unverified) |
| `FLASH_AND_DUMP.md` | SWD flash/dump reference (Black Magic Probe + ST-Link v2/OpenOCD) |

## Related Repositories

- [ACEPRO-katapult-bootloader](https://github.com/Kobra-S1/ACEPRO-katapult-bootloader) — Install Katapult via stock OTA (no SWD required)
- [Klipper](https://github.com/Klipper3d/klipper) — Upstream Klipper firmware
- [Katapult](https://github.com/Arksine/katapult) — USB/CAN bootloader for Klipper

## License

GPL-3.0
